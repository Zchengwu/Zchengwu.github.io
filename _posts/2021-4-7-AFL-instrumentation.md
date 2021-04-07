# AFL 插桩代码的运行机制

> 通过阅读源码系统理解桩代码的工作机制



## afl-gcc/g++

使用AFL，首先需要通过 `afl-gcc/afl-clang` 等工具来编译目标，在这个过程中会对其进行插桩。

以 `afl-gcc` 为例，`afl-gcc` 实际上是 `gcc` 的wrapper。

可以添加一些输出，从而在调用 `execvp()` 之前打印全部命令行参数，看看 `afl-gcc` 所执行的具体指令。

```c
// afl-gcc.c
int main(int argc, char** argv) {
  ...
  /*
  for(int i = 0; i < cc_par_cnt; i++)
  printf("%s ", cc_params[i]);
  printf("\n");
  */
  execvp(cc_params[0], (char**)cc_params);
  ...
}

$ afl-gcc
...
gcc -O3 -funroll-loops -Wall -D_FORTIFY_SOURCE=2 -g -Wno-pointer-sign -DAFL_PATH="/usr/local/lib/afl" -DDOC_PATH="/usr/local/share/doc/afl" -DBIN_PATH="/usr/local/bin" test-instr.c -o test-instr -ldl -B . -g -O3 -funroll-loops -D__AFL_COMPILER=1 -DFUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION=1
...
```

可以看到具体指令中，`afl-gcc` 具体调用 `gcc`，并定义了一些宏，其中最重要的就是 `-B .` 这条。由`gcc --help`可知，`-B` 用于设置编译器的搜索路径，这里便是默认设置成 `.`。(因为我没有设置的环境变量 `AFL_PATH` 的值(代表AFL路径)；如果`make install`，`-B`的参数还会不一样)。

> 如果了解编译过程，那么就知道把源代码编译成二进制，主要是经过`源代码 => 汇编代码 => 二进制`这样的过程。而将汇编代码编译成为二进制的工具，即为汇编器 assembler。Linux系统下的常用汇编器是 `/usr/bin/as`。
>

不过，编译完成AFL后，在其目录下也会存在一个 `as` 文件，并作为符号链接指向当前目录下的 `afl-as`。所以，如果通过 `-B` 选项为gcc设置了搜索路径，那么 `afl-as` 便会作为汇编器，执行实际的汇编操作。AFL的**代码插桩**，就是在将源文件编译为汇编代码后，通过 `afl-as` 完成。



## afl-as

继续阅读文件 `afl-as.c`，其大致逻辑是处理汇编代码，在分支处插入桩代码，并最终再调用 `as` 进行真正的汇编。具体插入代码的部分如下：

```c
fprintf(outf, use_64bit ? trampoline_fmt_64 : trampoline_fmt_32, R(MAP_SIZE));
```

通过 `fprintf` 将格式化字符串添加到汇编文件的对应位置。这里跟着文档看到32位的具体桩代码，即 `trampoline_fmt_32` 的内容：

- 保存寄存器内容
- 将 `ecx` 的值设置为 `fprintf()` 所要打印的变量内容 <= `R(MAP_SIZE)`
  - 全局搜索 `R(x)` 的定义: `(random() % (x))`，所以 `R(MAP_SIZE)` 即为0到MAP_SIZE之间的一个随机数。
- 调用方法 `__afl_maybe_log()`
- 恢复寄存器内容

```
static const u8* trampoline_fmt_32 =

  "\n"
  "/* --- AFL TRAMPOLINE (32-BIT) --- */\n"
  "\n"
  ".align 4\n"
  "\n"
  "leal -16(%%esp), %%esp\n"
  "movl %%edi,  0(%%esp)\n"
  "movl %%edx,  4(%%esp)\n"
  "movl %%ecx,  8(%%esp)\n"
  "movl %%eax, 12(%%esp)\n"
  "movl $0x%08x, %%ecx\n"
  "call __afl_maybe_log\n"
  "movl 12(%%esp), %%eax\n"
  "movl  8(%%esp), %%ecx\n"
  "movl  4(%%esp), %%edx\n"
  "movl  0(%%esp), %%edi\n"
  "leal 16(%%esp), %%esp\n"
  "\n"
  "/* --- END --- */\n"
  "\n";
```

`__afl_maybe_log`作为插桩代码所执行的实际内容，会在接下来详细展开，这里我们只分析`"movl $0x%08x, %%ecx\n"`这条指令。

因此，在处理到某个分支，需要插入桩代码时，`afl-as`会生成一个随机数，作为运行时保存在`ecx`中的值。而这个随机数，便是用于标识这个分支的key。在下面我们会详细介绍这个key是如何被使用的。



## fork server

在编译target(插桩过目标binary)之后，就开始通过 `afl-fuzz` 进行fuzzing。其大致思路是，通过不断mutate输入并喂给target，检查target运行是否崩溃。这其中就涉及到大量的fork和执行target的操作。

为了高效地完成上述重复操作，AFL实现了一套 fork server机制，其基本思路就是：Fuzzer在fork一个target进程后，target会运行一个fork server；Fuzzer不负责fork大量target子进程，而是负责与fork server通信，获取target子进程运行结果等信息，由fork server负责fork并继续执行target子进程的任务。

<img src="/Users/wzc/Library/Application Support/typora-user-images/image-20210301222543206.png" alt="image-20210301222543206" style="zoom:50%;" />

> 这样设计的最大好处是被fuzz的target程序只需要经历一次execve()调用，链接和libc初始化等操作。

启动fork server的逻辑作为桩代码的一部分插入在target中，在第一次进入`__afl_maybe_log`中启动fork server下面看下代码中`fork server`的具体实现。首先，fuzzer执行`fork()`得到父进程和子进程，这里的父进程仍然为fuzzer，子进程则为target进程，即将来的fork server。

```c
forksrv_pid = fork();
```

fuzzer和fork server之间使用管道通信，具体使用了2个管道，一个用于传递状态(status)，另一个用于传递命令(ctrl)：

```c
int st_pipe[2], ctl_pipe[2];
```

对于子进程fork server，会做很多准备工作，包括把两个管道的一端分配到预先指定的fd(FORKSRV_FD, FORKSRV_FD+1)，并最终执行target。

```c
if (dup2(ctl_pipe[0], FORKSRV_FD) < 0) PFATAL("dup2() failed");
if (dup2(st_pipe[1], FORKSRV_FD + 1) < 0) PFATAL("dup2() failed");
...
execv(target_path, argv);
```

之后父进程fuzzer继续执行，通过管道读取来自fork server发来的信息，如果是一个四字节长的正常信息，则表示fork server创建完成。

```
fsrv_st_fd  = st_pipe[0];
...
rlen = read(fsrv_st_fd, &status, 4);
...
if (rlen == 4) {
  OKF("All right - fork server is up.");
  return;
}
```

接下来，我们来分析fork server是如何与fuzzer通信的。

fork server侧的具体操作，也是在之前提到的方法 `__afl_maybe_log()` 中。首先，通过写入状态管道`st_pipe[1]`，fork server会通知fuzzer，其已经准备完毕，可以开始fork了，而这正是上面提到的父进程等待的信息：

```
	"__afl_forkserver:\n"
  "\n"
  "  /* Enter the fork server mode to avoid the overhead of execve() calls. */\n"
  "\n"
  "  pushl %eax\n"
  "  pushl %ecx\n"
  "  pushl %edx\n"
  "\n"
  "  /* Phone home and tell the parent that we're OK. (Note that signals with\n"
  "     no SA_RESTART will mess it up). If this fails, assume that the fd is\n"
  "     closed because we were execve()d from an instrumented binary, or because\n" 
  "     the parent doesn't want to use the fork server. */\n"
  "\n"
  "  pushl $4          /* length    */\n"
  "  pushl $__afl_temp /* data      */\n"
  "  pushl $" STRINGIFY((FORKSRV_FD + 1)) "  /* file desc */\n"
  "  call  write\n"
  "  addl  $12, %esp\n"
  "\n"
  "  cmpl  $4, %eax\n"
  "  jne   __afl_fork_resume\n"
```

接下来，fork server进入等待状态`__afl_fork_wait_loop`，读取命令管道`ctl_pipe[0]`，直到fuzzer通知其开始fork：

```
	"__afl_fork_wait_loop:\n"
  "\n"
  "  /* Wait for parent by reading from the pipe. Abort if read fails. */\n"
  "\n"
  "  pushl $4          /* length    */\n"
  "  pushl $__afl_temp /* data      */\n"
  "  pushl $" STRINGIFY(FORKSRV_FD) "        /* file desc */\n"
  "  call  read\n"
  "  addl  $12, %esp\n"
  "\n"
  "  cmpl  $4, %eax\n"
  "  jne   __afl_die\n"
```

一旦fork server接收到fuzzer的信息，便调用`fork()`，得到父进程和子进程：

```
	"  call fork\n"
  "\n"
  "  cmpl $0, %eax\n"
  "  jl   __afl_die\n"
  "  je   __afl_fork_resume\n"
```

子进程是实际执行target的进程，其跳转到`__afl_fork_resume`。在这里会关闭不再需要的管道，并继续执行：

```
	"__afl_fork_resume:\n"
  "\n"
  "  /* In child process: close fds, resume execution. */\n"
  "\n"
  "  pushl $" STRINGIFY(FORKSRV_FD) "\n"
  "  call  close\n"
  "\n"
  "  pushl $" STRINGIFY((FORKSRV_FD + 1)) "\n"
  "  call  close\n"
  "\n"
  "  addl  $8, %esp\n"
  "\n"
  "  popl %edx\n"
  "  popl %ecx\n"
  "  popl %eax\n"
  "  jmp  __afl_store\n"
```

父进程则仍然作为fork server运行，其会将子进程的pid通过状态管道发送给fuzzer，并等待子进程执行完毕；一旦子进程执行完毕，则再通过状态管道，将其结束状态发送给fuzzer；之后再次进入等待状态`__afl_fork_wait_loop`：

```
	"  /* In parent process: write PID to pipe, then wait for child. */\n"
  "\n"
  "  movl  %eax, __afl_fork_pid\n"
  "\n"
  "  pushl $4              /* length    */\n"
  "  pushl $__afl_fork_pid /* data      */\n"
  "  pushl $" STRINGIFY((FORKSRV_FD + 1)) "      /* file desc */\n"
  "  call  write\n"
  "  addl  $12, %esp\n"
  "\n"
  "  pushl $0             /* no flags  */\n"
  "  pushl $__afl_temp    /* status    */\n"
  "  pushl __afl_fork_pid /* PID       */\n"
  "  call  waitpid\n"
  "  addl  $12, %esp\n"
  "\n"
  "  cmpl  $0, %eax\n"
  "  jle   __afl_die\n"
  "\n"
  "  /* Relay wait status to pipe, then loop back. */\n"
  "\n"
  "  pushl $4          /* length    */\n"
  "  pushl $__afl_temp /* data      */\n"
  "  pushl $" STRINGIFY((FORKSRV_FD + 1)) "  /* file desc */\n"
  "  call  write\n"
  "  addl  $12, %esp\n"
  "\n"
  "  jmp __afl_fork_wait_loop\n"
```

以上就是fork server的主要逻辑，现在我们再回到fuzzer侧。在fork server启动完成后，一旦需要执行某个测试用例，则fuzzer会调用 `run_target()` 方法。在此方法中，便是通过命令管道，通知fork server准备fork；并通过状态管道，获取子进程pid：

```
	if ((res = write(fsrv_ctl_fd, &prev_timed_out, 4)) != 4) {
    if (stop_soon) return 0;
    RPFATAL(res, "Unable to request new process from fork server (OOM?)");
  }

  if ((res = read(fsrv_st_fd, &child_pid, 4)) != 4) {
  	if (stop_soon) return 0;
  	RPFATAL(res, "Unable to request new process from fork server (OOM?)");
  }
```

随后，fuzzer再次读取状态管道，获取子进程退出状态，并由此来判断子进程结束的原因，例如正常退出、超时、崩溃等，并进行相应的记录。

```c
	if ((res = read(fsrv_st_fd, &status, 4)) != 4) {
	...
	
	if (WIFSIGNALED(status) && !stop_soon) {
    kill_signal = WTERMSIG(status);
    if (child_timed_out && kill_signal == SIGKILL) return FAULT_TMOUT;
    return FAULT_CRASH;
  }
```



## shared memory

> 作为fuzzer，AFL并不是无脑地随机变化(虽然也支持这种模式，即dumb模式)，其最大的特点就是会对target进行插桩，手机code coverage信息，以辅助mutated input生成。具体地，插桩后的target，会记录执行过程的分支信息；随后，fuzzer便可根据这些信息，判断这次执行整体流程和代码覆盖情况。

AFL使用共享内存，来完成以上信息在fuzzer和target之间的传递。具体地，fuzzer在启动时，会执行 `setup_shm()` 方法进行配置。其首先调用 `shemget()` 分配一块共享内存，大小 `MAP_SIZE` 为64K:

```c
// 分配一块大小为MAP_SIZE(64K)的共享内存
shm_id = shmget(IPC_PRIVATE, MAP_SIZE, IPC_CREAT | IPC_EXCL | 0600);
```

分配成功后，该共享内存的标志符会被设置到**环境变量**中，从而之后 `fork()` 得到的子进程可以通过该环境变量，得到这块共享内存的标志符：

```c
shm_str = alloc_printf("%d", shm_id);
if (!dumb_mode) setenv(SHM_ENV_VAR, shm_str, 1);
```

并且，fuzzer本身，会使用变量`trace_bits` 来保存共享内存的地址：

```c
// fuzzer本身使用变量trace bits保留共享内存的地址
trace_bits = shmat(shm_id, NULL, 0);
```

在每次target执行之前，fuzzer首先将该共享内容清零：

```c
memset(trace_bits, 0, MAP_SIZE); 
```

接下来，我们再来看看target是如何获取并使用这块共享内存的。相关代码同样也在上面提到的方法 `__afl_maybe_log()` 中。首先，会检查是否已经将共享内存映射完成：

```
	" /* Check if SHM region is already mapped. */\n"
  "\n"
  " movl __afl_area_ptr, %edx\n"
  " testl %edx, %edx\n"
  " je __afl_setup\n"
```

`__afl_area_ptr`中保存的就是共享内存映射到target的内存空间中的地址，如果其不是NULL，便保存在`edx`中继续执行；否则进一步跳转到`__afl_setup`。

`__afl_setup`处会做一些错误检查，然后获取环境变量`AFL_SHM_ENV`的内容并将其转为整型。查看其定义便可知，这里获取到的，便是之前fuzzer保存的共享内存的标志符。

```
  "__afl_setup:\n"
  "\n"
  " /* Do not retry setup if we had previous failures. */\n"
  "\n"
  " cmpb $0, __afl_setup_failure\n"
  " jne __afl_return\n"
  "\n"
  " /* Map SHM, jumping to __afl_setup_abort if something goes wrong.\n"
  " We do not save FPU/MMX/SSE registers here, but hopefully, nobody\n"
  " will notice this early in the game. */\n"
  "\n"
  " pushl %eax\n"
  " pushl %ecx\n"
  "\n"
  " pushl $.AFL_SHM_ENV\n"
  " call getenv\n"
  " addl $4, %esp\n"
  "\n"
  " testl %eax, %eax\n"
  " je __afl_setup_abort\n"
  "\n"
  " pushl %eax\n"
  " call atoi\n"
  " addl $4, %esp\n"
```

最后，通过调用`shmat()`，target将这块共享内存也映射到了自己的内存空间中，并将其地址保存在`__afl_area_ptr`及`edx`中。由此，便完成了fuzzer与target之间共享内存的设置。

```
  " pushl $0 /* shmat flags */\n"
  " pushl $0 /* requested addr */\n"
  " pushl %eax /* SHM ID */\n"
  " call shmat\n"
  " addl $12, %esp\n"
  "\n"
  " cmpl $-1, %eax\n"
  " je __afl_setup_abort\n"
  "\n"
  " /* Store the address of the SHM region. */\n"
  "\n"
  " movl %eax, __afl_area_ptr\n"
  " movl %eax, %edx\n"
  "\n"
  " popl %ecx\n"
  " popl %eax\n"
```

> 如果使用了fork server模式，那么上述获取共享内存的操作，是在fork server中进行；随后fork出来的子进程，只需直接使用这个共享内存即可。



## 分支信息的记录

看白皮书可知，AFL是根据二元tuple(跳转的源地址和目标地址)来记录分支信息，从而获取target的执行流程和代码覆盖情况，其伪代码如下：

```
cur_location = <COMPILE_TIME_RANDOM>;
shared_mem[cur_location ^ prev_location]++; 
prev_location = cur_location >> 1;
```

我们再回到方法`__afl_maybe_log()`中。上面提到，在target完成准备工作后，共享内存的地址被保存在寄存器`edx`中。随后执行以下代码：

```
  "__afl_store:\n"
  "\n"
  " /* Calculate and store hit for the code location specified in ecx. There\n"
  " is a double-XOR way of doing this without tainting another register,\n"
  " and we use it on 64-bit systems; but it's slower for 32-bit ones. */\n"
  "\n"
#ifndef COVERAGE_ONLY 
  " movl __afl_prev_loc, %edi\n"
  " xorl %ecx, %edi\n"
  " shrl $1, %ecx\n"
  " movl %ecx, __afl_prev_loc\n"
#else 
  " movl %ecx, %edi\n"
#endif /* ^!COVERAGE_ONLY */ 
  "\n"
#ifdef SKIP_COUNTS 
  " orb $1, (%edx, %edi, 1)\n"
#else 
  " incb (%edx, %edi, 1)\n"
```

这里对应的便正是文档中的伪代码。具体地，变量`__afl_prev_loc`保存的是前一次跳转的”位置”，其值与`ecx(=R(MAXSIZE))`做异或后，保存在`edi`中，并以`edx`（共享内存）为基址，对`edi`下标处进行加一操作。而`ecx`的值右移1位后，保存在了变量`__afl_prev_loc`中。

那么，这里的`ecx`，保存的应该就是伪代码中的`cur_location`了。回忆之前介绍代码插桩的部分：

```
static const u8* trampoline_fmt_32 = 
...
  "movl $0x%08x, %%ecx\n"
  "call __afl_maybe_log\n"
```

在每个插桩处，afl-as会添加相应指令，将`ecx`的值设为0到MAP_SIZE之间的某个随机数，从而实现了伪代码中的`cur_location = <COMPILE_TIME_RANDOM>;`。

因此，AFL为每一桩代码位置生成一个随机数，作为其位置记录；随后，对分支处的”源位置“和”目标位置“进行异或，并将异或的结果作为该分支的key，保存每个分支的执行次数。用于保存执行次数的数据结构实际上是一个哈希表，大小为`MAP_SIZE=64K`，当然**会存在碰撞的问题**；但根据AFL文档中的介绍，对于不是很复杂的目标，碰撞概率还是可以接受的：

```
   Branch cnt | Colliding tuples | Example targets
  ------------+------------------+-----------------
        1,000 | 0.75%            | giflib, lzo
        2,000 | 1.5%             | zlib, tar, xz
        5,000 | 3.5%             | libpng, libwebp
       10,000 | 7%               | libxml
       20,000 | 14%              | sqlite
       50,000 | 30%              | -
```

如果一个目标过于复杂，那么AFL状态面板中的map_density信息就会有相应的提示：

```
┬─ map coverage ─┴───────────────────────┤
│    map density : 3.61% / 14.13%        │
│ count coverage : 6.35 bits/tuple       │
┼─ findings in depth ────────────────────┤
```

这里的map density，就是这张哈希表的密度。可以看到，上面示例中，该次执行的哈希表密度仅为3.61%，即整个哈希表差不多有95%的地方还是空的，所以碰撞的概率很小。不过，如果目标很复杂，map density很大，那么就需要考虑到碰撞的影响了。此种情况下的具体处理方式可见官方文档。

另外，比较有意思的是，AFL需要将`cur_location`右移1位后，再保存到`prev_location`中。官方文档中解释了这样做的原因。假设target中存在`A->A`和`B->B`这样两个跳转，如果不右移，那么这两个分支对应的异或后的key都是0，从而无法区分；另一个例子是`A->B`和`B->A`，如果不右移，这两个分支对应的异或后的key也是相同的。

由上述分析可知，共享内存实际上是一张哈希表，用于记录不同输入在target上执行过程中每个分支的执行数量。随后，当target执行结束后，fuzzer便开始对这张表进行分析，从而判断代码的执行情况。



## 分支信息预处理

首先，fuzzer对`trace_bits`（共享内存）进行预处理：

```
classify_counts((u32*)trace_bits);
```

具体地，target是将每个分支的执行次数用1个byte来储存，而fuzzer则进一步把这个执行次数归入以下的buckets中：

```
static const u8 count_class_lookup8[256] = {

  [0]           = 0, 
  [1]           = 1, 
  [2]           = 2, 
  [3]           = 4, 
  [4 ... 7]     = 8, 
  [8 ... 15]    = 16,
  [16 ... 31]   = 32,
  [32 ... 127]  = 64,
  [128 ... 255] = 128

};
```

举个例子，如果某分支执行了1次，那么落入第2个bucket，其计数byte仍为1；如果某分支执行了4次，那么落入第5个bucket，其计数byte将变为8，等等。

这样处理之后，对分支执行次数就会有一个简单的归类。例如，如果对某个测试用例处理时，分支A执行了32次；对另外一个测试用例，分支A执行了33次，那么AFL就会认为这两次的代码覆盖是相同的。当然，这样的简单分类肯定不能区分所有的情况，不过**在某种程度上，处理了一些因为循环次数的微小区别**，而误判为不同执行结果的情况。

具体如何使用到trace_bits见AFL内部实现细节



## 参考

[AFL内部实现细节小记](http://rk700.github.io/2017/12/28/afl-internals/)

---

### __afl_maybe_log 反编译

直接放一个反编译出来的代码方便理解上述内容

```c
char __usercall _afl_maybe_log@<al>(char a1@<of>, __int64 a2@<rcx>, ...)
{
  ...

  v18 = a1;
  v19 = _afl_area_ptr;
  if ( !_afl_area_ptr )
  {
    if ( _afl_setup_failure )
      return v18 + 127;
    v19 = _afl_global_area_ptr;
    if ( _afl_global_area_ptr )
    {
      _afl_area_ptr = _afl_global_area_ptr;
    }
    else
    {
      ...
      v22 = getenv("__AFL_SHM_ID");
      if ( !v22 || (v23 = atoi(v22), v24 = shmat(v23, 0LL, 0), v24 == (void *)-1LL) )
      {
        ++_afl_setup_failure;
        v18 = v29;
        return v18 + 127;
      }
      _afl_area_ptr = (__int64)v24;
      _afl_global_area_ptr = v24;
      v28 = (__int64)v24;
      if ( write(199, &_afl_temp, 4uLL) == 4 )
      {
        while ( 1 )
        {
          v25 = 198;
          if ( read(198, &_afl_temp, 4uLL) != 4 )
            break;
          LODWORD(v26) = fork();
          if ( v26 < 0 )
            break;
          if ( !v26 )
            goto __afl_fork_resume;
          _afl_fork_pid = v26;
          write(199, &_afl_fork_pid, 4uLL);
          v25 = _afl_fork_pid;
          LODWORD(v27) = waitpid(_afl_fork_pid, &_afl_temp, 0);
          if ( v27 <= 0 )
            break;
          write(199, &_afl_temp, 4uLL);
        }
        _exit(v25);
      }
__afl_fork_resume:
      close(198);
      close(199);
      v19 = v28;
      v18 = v29;
      a2 = v30;
    }
  }
  v20 = _afl_prev_loc ^ a2; // <= 之后fork出的子进程target就运行这里的代码就行了
  _afl_prev_loc ^= v20;
  _afl_prev_loc = (unsigned __int64)_afl_prev_loc >> 1;
  ++*(_BYTE *)(v19 + v20);
  return v18 + 127;
}
```





