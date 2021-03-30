---
layout:     post
title:      "# AFL 简单测试 && 问题记录"
subtitle:   ""
date:       2021-3-22
author:     "Ricky Wu"
header-img: "img/post-bg.jpg"
tags:
    - AFL, Fuzzing
---

## 1. AFL 安装

```
wget https://lcamtuf.coredump.cx/afl/releases/afl-latest.tgz
tar -xvf ./afl-latest.tgz
cd afl-2.52b
make && sudo make install
```

## 2. fuzz简单程序

```c
// test1.c
#include <stdio.h> 
int main(int argc, char *argv[])
{
    char buf[100]={0};
    gets(buf);		// format string
    printf(buf);	// stack overflow
    return 0;
}

// afl-gcc -o test1 test1.c 插桩编译
```

创建`fuzz_in`，`fuzz-out`文件夹

<img src="https://wwzzcccccc.me/img/post/AFL_Helloworld/image-1.png" style="zoom:50%;" />

开始fuzz

```
afl-fuzz -i ./fuzz_in -o ./fuzz_out ./test
```

运行界面

<img src="https://wwzzcccccc.me/img/post/AFL_Helloworld/image-2.png" style="zoom:50%;" />

crash分析

- 分析运行结果中的连个uniq crashes

  <img src="https://wwzzcccccc.me/img/post/AFL_Helloworld/image-3.png" style="zoom:50%;" />

### tips

> 想要继续之前的afl，使用之前相同的命令，然后把`-i`的输入文件夹名改为`-`
>
> ```
> afl-fuzz -i - -o ./fuzz_out ./test
> ```

### 问题记录

#### 1. core pattern

​	切换到root，输入`echo core > /proc/sys/kernel/core_pattern`

<img src="https://wwzzcccccc.me/img/post/AFL_Helloworld/image-4.png" style="zoom:50%;" />

#### 2. no input case

​	在`fuzz_in`文件夹放一个输入，aaa。`echo aaa > ./fuzz_in/aaa`

<img src="https://wwzzcccccc.me/img/post/AFL_Helloworld/image-5.png" style="zoom:50%;" />

---

[其他错误](https://blog.csdn.net/weixin_48505549/article/details/110945509?spm=1001.2014.3001.5501)

- memory limit
- cpu frequency scaling