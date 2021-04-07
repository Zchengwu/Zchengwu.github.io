---
layout:     post
title:      "AFL bitmap更新细节"
subtitle:   ""
date:       2021-4-7
author:     "Ricky Wu"
header-img: "img/post-bg.jpg"
tags:
    - AFL
    - Fuzzing
---

## 几个bitmap

> - `trace_bits` 记录当前的tuple信息；
> - `virgin_bits` 用来记录总的tuple信息；
> - `virgin_tmout` 记录fuzz过程中出现的所有目标程序的timeout时的tuple信息；
> - `virgin_crash` 记录fuzz过程中出现的crash时的tuple信息；


## 载入指定bitmap

- `-B` 参数，制定载入的bitmap

  ```c
  			...
  			case 'B': /* load bitmap */
  
          /* This is a secret undocumented option! It is useful if you find
             an interesting test case during a normal fuzzing process, and want
             to mutate it without rediscovering any of the test cases already
             found during an earlier run.
  
             To use this mode, you need to point -B to the fuzz_bitmap produced
             by an earlier run for the exact same binary... and that's it.
  
             I only used this once or twice to get variants of a particular
             file, so I'm not making this an official setting. */
  
          if (in_bitmap) FATAL("Multiple -B options not supported");
  
          in_bitmap = optarg;
          read_bitmap(in_bitmap);
          break;
        ...
  ```

  `read_bitmap` 函数将指定bitmap加载进 `virgin_bits` 中。

  ```c
  /* Read bitmap from file. This is for the -B option again. */
  EXP_ST void read_bitmap(u8* fname) {
    s32 fd = open(fname, O_RDONLY);
    if (fd < 0) PFATAL("Unable to open '%s'", fname);
    ck_read(fd, virgin_bits, MAP_SIZE, fname);
    close(fd);
  }
  ```



## fork server 更新 trace_bits

见[AFL 插桩代码运行机制](AFL 插桩代码运行机制(完).md) 。



## bitmap相关函数

### setup_shm

- `setup_shm` 设置共享内存，用 0xff 初始化 `virgin_bits`，`virgin_tmout`， `virgin_crash`(如果设置`-B` 就不初始化`virgin_bits`)；同时会申请一块共享内存，fuzzer通过`trace_bits`变量保存共享内存的地址，fork server和它的子进程则通过环境变量`SHM_ENV_VAR`来访问共享内存。

  ```c
  EXP_ST void setup_shm(void) {
  	...
  	if (!in_bitmap) memset(virgin_bits, 255, MAP_SIZE);
    memset(virgin_tmout, 255, MAP_SIZE);
    memset(virgin_crash, 255, MAP_SIZE);
  	...
  	if (!dumb_mode) setenv(SHM_ENV_VAR, shm_str, 1);
  	...
  	trace_bits = shmat(shm_id, NULL, 0);
  	...
  }
  ```



### has_new_bits

- 64位下，初始化current和virgin为trace_bits和virgin_map的u64首元素地址，初始化ret为0

- 以8个字节一组遍历trace_bits和virgin_map

  - 如果current不为所动0，且`current&virgin`不为0，代表在这8个字节代表的edge上出现初次访问或者访问次数发生变化

    - `==`优先级大于`&&`，如果出现新的edge访问，ret=2；如果只是edge访问次数变化，ret=1

      ```
      (cur[0] && vir[0] == 0xff) || (cur[1] && vir[1] == 0xff) ||
      (cur[2] && vir[2] == 0xff) || (cur[3] && vir[3] == 0xff) ||
      (cur[4] && vir[4] == 0xff) || (cur[5] && vir[5] == 0xff) ||
      (cur[6] && vir[6] == 0xff) || (cur[7] && vir[7] == 0xff)
      ```

    - `*virgin &= ~*current;`，更新virgin_map ??
    
  - 如果ret部位0，且传入的virgin_map是virgin_bits，bitmap_changed=1，做标记

  - 返回ret





### classify_counts

> 在每次run_target之后，会进行一次classify_counts操作，目的是把bitmap里每个字节代表的执行次数归入以下的buckets
>
> ```
> [0]           = 0, 
> [1]           = 1, 
> [2]           = 2, 
> [3]           = 4, 
> [4 ... 7]     = 8, 
> [8 ... 15]    = 16,
> [16 ... 31]   = 32,
> [32 ... 127]  = 64,
> [128 ... 255] = 128
> ```

- 8个字节为单位遍历bitmap
  - 以2个字节为单位，在`count_class_lookup16`找到每个字节对应的bytes，替换



### update_bitmap_score

> 当我们执行完一次target并通过classify_counts将执行次数归好类之后，我们检查当前这个testcase对于trace_bits中所有访问到的edge是不是一个更优的选择，如果是的话进行一个替换。
>
> 最终所有的edge都会有自己的testcase，这些testcase构成favorable集合，所谓favorable就是说这些testcase能够覆盖所有的目前访问过的edge，并且他们的大小更小，执行时间更短，测试他们的效率更高。在下一轮fuzz的时候我们专注于在favorable集合上进行fuzz。

- 首先计算出这个case的fav_factor，`fav_factor = q->exec_us * q->len`，执行时间乘以样例大小，作为这个case的衡量指标

- 遍历trace_bits数组的所有字节，如果不为0表示它代表的edge被这个case访问到

  - 如果这个edge的top_rated存在，标示先前已经有别的case访问过这个路径了，我们将两者做个比较

    - 比的就是上面这个factor，即执行时间和样例大小成绩。越小越好

    - 如果当前case更优

      - 原先的case的tc_ref字段减一，如果减为0就释放它的trace_mini内存空间并把指针字段设置为0

        ```
        if (!--top_rated[i]->tc_ref) {
          ck_free(top_rated[i]->trace_mini);
          top_rated[i]->trace_mini = 0;
        }
        ```

      - 设置`top_rated[i]`为当前case，然后将这个case的tc_ref字段加1

      - 如果这个case的trace_mini字段指针为空，分配空间，则将trace_bits经过minimize_bits压缩，然后存到trace_mini字段里

        ```
        if (!q->trace_mini) {
          q->trace_mini = ck_alloc(MAP_SIZE >> 3);
          minimize_bits(q->trace_mini, trace_bits);
        }
        ```



### cull_queue

>  精简队列

- 如果score_changed为0，表示bitmap没有变化，或者是dumb_mode，直接返回

- 设置score_changed=0

- 初始化temp_v数组，大小为 `MAPSIZE / 8`，设置其所有byte的初始值为0xff

- `queued_favored = 0;  pending_favored = 0;`

- 开始遍历queue队列，设置所有case的favored为0

- 将i从0到MAP_SIZE遍历，遍历top_rated中的case，选出一组queue entry，即testcase，使它们能够覆盖到现在所有发现的edge，而且这个集合里的元素要尽可能的少。(贪心)

  - 这又是个不好懂的位运算，`temp_v[i >> 3] & (1 << (i & 7))`与上面的差不多，中间的或运算改成了与，是为了检查该位是不是0，即判断该path对应的bit有没有被置位。

  ```
  if (top_rated[i] && (temp_v[i >> 3] & (1 << (i & 7)))) 
  ```

  - 如果`top_rated[i]`有值，且该path在temp_v里被置位

    - 就从temp_v中清除掉所有`top_rated[i]`覆盖到的path，将对应的bit置为0

      ```
      temp_v[j] &= ~top_rated[i]->trace_mini[j];
      ```

    - 设置`top_rated[i]->favored`为1，queued_favored计数器加一

      - favored字段主要在`fuzz_one()`中使用

        ```
        /* If we have any favored, non-fuzzed new arrivals in the queue,
        possibly skip to them at the expense of already-fuzzed or non-favored
        cases. */
        
        if ((queue_cur->was_fuzzed || !queue_cur->favored) &&
        	UR(100) < SKIP_TO_NEW_PROB) return 1;
        ```

    - 如果`top_rated[i]`的was_fuzzed字段是0，代表其还没有fuzz过，则将pending_favored计数器加一

  - 遍历queue队列

    - `mark_as_redundant(q, !q->favored)`

      - 也就是说，如果不是favored的case，就被标记成redundant_edges

      

---

### minimize_bits

> 通过bitmap压缩，把用一个字节记录的edge访问次数转化为用1bit记录这个edge是否访问到

```c
static void minimize_bits(u8* dst, u8* src) {
  u32 i = 0;
  while (i < MAP_SIZE) {
    if (*(src++)) dst[i >> 3] |= 1 << (i & 7);
    i++;
  }
}
```

