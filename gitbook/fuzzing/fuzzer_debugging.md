# Fuzzer Debugging

记录一些 debug Fuzzers 的技巧

## GDB 子进程调试

fuzzer 通常会 `fork()` 出子进程进行 fuzzing, 所以只调试主进程是不够的.

在调试子进程时, 一开始就设置跟踪子进程, 进入子进程以后手动加载符号表

```shell
# set gdb following child process 
set follow-fork-mode [parent|child]
# set off as detach from parent process
# set on as concurrently debug for parent and child process 
set detach-on-fork [off|on]

# switch parent and child and grandchild process
info inferior
inferior [1|2|...]

# if above not work, then can set the simbol manually
symbol-file /path/to/target_binary
```

遇到无法正确加载符号表的情况, 有两种解决方法

1. 修改源代码进行侵入式 `print` debug
2. 对着汇编调试, 再子进程和父进程之间反复横跳逐行代码/指令执行. 当遇到执行不下去的时候就 `inferior 1|2` 或者 `infer 1|2` 切换到另一个进程继续执行, 即正常的多进程调试方式.



## fork 超时

```shell
[-] PROGRAM ABORT : Timeout while initializing fork server (setting AFL_FORKSRV_INIT_TMOUT may help)
```

使用命令设置环境变量

```shell
# time: ms
export AFL_FORKSRV_INIT_TMOUT=100000
```




