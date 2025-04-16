# AFL++ Source Code Reading II - Modes

本文从主逻辑开始解析 AFL++ 源代码







[forkserver source code](https://github.com/AFLplusplus/AFLplusplus/blob/ea14f3fd40e32234989043a525e3853fcb33c1b6/src/afl-forkserver.c#L667)

```c
    if (dup2(ctl_pipe[0], FORKSRV_FD) < 0) { PFATAL("dup2() failed"); }
    if (dup2(st_pipe[1], FORKSRV_FD + 1) < 0) { PFATAL("dup2() failed"); }
```











## Reference

[1] https://github.com/AFLplusplus/AFLplusplus

[2] https://blog.ritsec.club/posts/afl-under-hood/
