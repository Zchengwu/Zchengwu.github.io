---
layout:     post
title:      "AFL mutation细节"
subtitle:   ""
date:       2021-3-30
author:     "Ricky Wu"
header-img: "img/post-bg.jpg"
tags:
    - AFL, Fuzzing
---

> 总体上AFL维护一个文件队列，每次从其中取出一个“最感兴趣的”，对它进行大量变异(6类)，然后检测target进程运行结果，是否崩溃、发现新路径等等。

所有文件变异操作都在`fuzz_one`里，总共大致分为6类：

- SIMPLE BITFLIP
- ARITHMETIC INC/DEC
- INTERESTING VALUES
- DICTIONARY STUFF
- HAVOC
- SPLICING

前四个合称`deterministic mutation`。



## mutation 细节

> 结合fuzz_one函数内部mutation部分代码

### SIMPLE BITFLIP

- 先来看一个宏

  ```c
  #define FLIP_BIT(_ar, _b) do { \
      u8* _arf = (u8*)(_ar); \
      u32 _bf = (_b); \
      _arf[(_bf) >> 3] ^= (128 >> ((_bf) & 7)); \
    } while (0)
  ```

  `_ar`指向out_buf，`_b`的范围是[0: len << 3)，`_bf & 7` 的取值为[0, 1, 2, 3, 4, 5, 6, 7]，在`_b`遍历[0: len << 3)，的过程中，一共会产生n组[0, ... , 7]，对input的每个bit进行翻转。

- `bitflip 1/1`(翻转长度/步长)， 对输入文件每个bit进行翻转，然后调用`common_fuzz_stuff`告诉fork server起一个新的子进程测试，最后再把这个bit翻转回来。

  ```c
  stage_max = len << 3;
  for (stage_cur = 0; stage_cur < stage_max; stage_cur++) {
      stage_cur_byte = stage_cur >> 3;
      FLIP_BIT(out_buf, stage_cur);
      if (common_fuzz_stuff(argv, out_buf, len)) goto abandon_entry;
      FLIP_BIT(out_buf, stage_cur);
   		...(原语识别)
   }
  ```

  - 在`bitflip 1/1`环节，还顺便完成了输入内部原语识别的工作。

    > 对于每个byte的最低位(least significant bit)翻转还进行了额外的处理：如果连续多个bytes的最低位被翻转后，程序的执行路径都未变化，而且与原始执行路径不一致，那么就把这一段连续的bytes判断是一条token。
    > 比如对于SQL的`SELECT *`，如果`SELECT`被破坏，则肯定和正确的路径不一致，而被破坏之后的路径却肯定是一样的，比如`AELECT`和`SBLECT`，显然都是无意义的，而只有不破坏token，才有可能出现和原始执行路径一样的结果，所以AFL在这里就是在猜解关键字token。 —— 摘自 sakura's blog

  - `bitflip 2/1`，类似，主要调整了循环数

    ```c
    stage_max   = (len << 3) - 1;
    for (stage_cur = 0; stage_cur < stage_max; stage_cur++) {
        stage_cur_byte = stage_cur >> 3;
        FLIP_BIT(out_buf, stage_cur);
        FLIP_BIT(out_buf, stage_cur + 1);
        if (common_fuzz_stuff(argv, out_buf, len)) goto abandon_entry;
        FLIP_BIT(out_buf, stage_cur);
        FLIP_BIT(out_buf, stage_cur + 1);
    }
    ```

  - 类似，`bitflip 4/1`，连续翻转4位

  - 在进行`bitflip 8/8`这一环节时，还生成了`effector map`，`effector map`始终贯穿在`deterministic fuzzing`环节。

    - 在对每个byte进行翻转时，如果trace_bits发生变化，即有新的路径产生，就将effector map中这个byte对应的位置设为1，标记为“有效”，否则标记为0；`eff_cnt++`
    - 根据queue_cur产生两种变化
      - 如果是dumb模式或者文件大小小于128，所有byte全部标记为1
      - 如果`eff_cnt`在effector map中占比超过90%，表明绝大多数byte都需要翻转，那么索性所有byte全部翻转；
    - `bitflip 8/8`，以byte为单位进行翻转，直接通过`0xff`与对应的byte进行异或，然后执行一次，记录结果
    - `bitflip 16/8`，设置`stage_max = len - 1`，以字为单位，通过异或`0xffff`进行翻转，然后执行一次，记录结果
      - 翻转前先检查eff_map，如果这两个字节在eff_map中都标记为0，则stage_max--，跳过翻转(因为它们是“无效的”)
    -  `bitflip 32/8`，设置`stage_max = len - 3，以双字为单位，通过异或`0xffffffff`进行翻转，然后执行一次，记录结果
      - 翻转前先检查eff_map，如果这四个字节在eff_map中都标记为0，则stage_max--，跳过翻转(因为它们是“无效的”)

    

### ARITHMETIC INC/DEC

- `arith8/8`，每次对一个byte进行加减操作变异(2)，然后各执行一次，记录结果
- `arith16/8`，每次对两个byte进行加减操作变异(4)，然后各执行一次，记录结果
- `arith32/8`，每次对四个byte进行加减操作变异(4)，然后各执行一次，记录结果
- 加减变异的上限为宏`#define ARITH_MAX 35`，会进行 `-1, +1, -2, +2, ..., -35, +35`算数变异；同时AFL对整数(两个byte以上)的大端序和小端序两种表示方式进行非别处理
- AFL还会只能地跳过ARITHMETIC的部分环节：
  - `arith8/8`， `arith16/8`， `arith32/8`，在算数变异后会通过`could_be_bitflip()`检查变异结果在bitflip环节是否出现过，如果是则跳过。
  -  `arith16/8`， `arith32/8`，检查要变异的byte在eff_map是不是被标记为“无效”，如果是则跳过



### INTERESTING VALUES

- `interest 8/8`，每次对8个bit，按照8个bit的步长从头开始，即对文件的每个byte进行替换

- `interest 16/8`，每次对8个bit，按照8个bit的步长从头开始，即对文件的每个word进行替换

- `interest 32/8`，每次对8个bit，按照8个bit的步长从头开始，即对文件的每个dword进行替换

  - ```c
    static s8  interesting_8[]  = { INTERESTING_8 };
    static s16 interesting_16[] = { INTERESTING_8, INTERESTING_16 };
    static s32 interesting_32[] = { INTERESTING_8, INTERESTING_16, INTERESTING_32 };
    ```



### DICTIONARY STUFF

`deterministic fuzzing`的最后一步

- user extras(over)，从头开始，将用户提供的tokens依次替换到原文件中，stage_max为`extras_cnt * len`
- user extras(insert)，从头开始，将用户提供的tokens依次插入到原文件中，stage_max为`extras_cnt * len`
- auto extras(over)，从头开始，将自动检测的tokens依次替换到原文件中，stage_max为`MIN(a_extras_cnt, USE_AUTO_EXTRAS) * len`
- 其中，用户提供的tokens，是在词典文件中设置并通过-x选项指定的，如果没有则跳过相应的子阶段。

- 如果queue_cur的`passed_det`为0，通过`mark_as_det_done()`进行标记




### HAVOC

16种随机变异方式：

- 随机对某一个bit进行翻转
- 随机选择某个byte为interesting value
- 随机选择某个word，并随机选择大、小端序，并将其设置为随机的interesting value
- 随机选择某个dword，并随机选择大、小端序，并将其设置为随机的interesting value
- 随机选择某个byte进行减去一个在`ARITH_MAX`范围内的随机数值
- 随机选择某个byte进行加上一个在`ARITH_MAX`范围内的随机数值
- 随机选择某个word，并随机选择大、小端序，减去一个在`ARITH_MAX`范围内的随机数值
- 随机选择某个word，并随机选择大、小端序，加上一个在`ARITH_MAX`范围内的随机数值
- 随机选择某个dword，并随机选择大、小端序，减去一个在`ARITH_MAX`范围内的随机数值
- 随机选择某个dword，并随机选择大、小端序，加上一个在`ARITH_MAX`范围内的随机数值
- 随机选取某个byte，并设置为随机数
- 随机删除某段bytes
- 随机选取一个位置，插入一段随机长度的内容，其中75%的概率是插入原文中随机位置的内容，25%概率是插入一段随机选取的内容。(choose_block_len??)
- 随机选取一个位置，替换一段随机长度的内容，其中75%的概率是插入原文中随机位置的内容，25%概率是插入一段随机选取的内容。(choose_block_len??)
- 随机选取一个位置，用随机选取的token（用户提供或自动生成的）替换
- 随机选取一个位置，用随机选取的token（用户提供或自动生成的）插入
- 同时，AFL会生成一个随机数，作为变异组合的数量，并根据这个数量，每次从上面那些方式中随机选取一个对输入文件进行变异。

> 如此这般丧心病狂的变异，原文件就大概率面目全非了，而这么多的随机性，也就成了fuzzing过程中的不可控因素，即所谓的“看天吃饭”了。 —— sakura's blog



### SPLICING

- 这个环节将从queue中选取一个其他的queue_entry，在一个随机的offset与queue_cur进行拼接。拼接结果送入havoc进行变异执行。
- 如果queue_cur通过了评估并且was_fuzzed字段为0的话，就标记`queue_cur->was_fuzzed = 1`，同时 pending_not_fuzzed -1

- 如果`queue_cur->favored`，则`pending_favored--`

