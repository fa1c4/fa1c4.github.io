# Miscellaneous Bugs Summary

记录写大型项目时 (LoC > 10k) 遇到的各种 bugs / 坑

## undefined reference to xx

```shell
/usr/bin/ld: /tmp/ccBrKxhX.ltrans2.ltrans.o: in function `fuzz_one_original':
/home/fa1c4/Desktop/funafl/src/afl-fuzz-one.c:3325: undefined reference to `UR'
/usr/bin/ld: /home/fa1c4/Desktop/funafl/src/afl-fuzz-one.c:3325: undefined reference to `UR'
collect2: error: ld returned 1 exit status
```

这种情况根因通常包括

+ 未定义函数
+ 未包含对应头文件
+ 对应头文件没加入链接目标
+ 使用 static / inline 声明的函数无法跨源文件引用



## headers havoc

C/C++ 语言开发大型项目时, 必定会碰到的问题有头文件互相包含, 比如在 `afl-fuzz-json.h` 中需要引用 `list.h` 的结构体, `afl-fuzz.h` 中又同时需要 `afl-fuzz-json.h` 和 `list.h` 的结构体/函数定义, 那么直接在头文件里互相包含, 即使增加 `#ifndef HEADER_XXX` 进行包含保护, 也会出错. 

一种 "不差" 实践: 只在 `.c` 源文件包含非自身的 `.h` 头文件, 避免非标准头文件 `.h` 之间互相包含, 如果 `.h` 里的声明需要用到其他 `.h` 里定义的变量/结构体等, 则使用前向声明, 只声明同名变量而不定义. 方式如下

```c
/* ----- afl-fuzz-json.h ----- */
...
// need struct afl_state to declare functions
struct afl_state; // which defined at afl-fuzz.c
...
void add_bb_count_key(struct afl_state* afl, int bb_random_val);    
...
/* ----- afl-fuzz-json.h ----- */

/* ----- afl-fuzz-json.c ----- */
#include "afl-fuzz-json.h"
#include "afl-fuzz.h"
    
void add_bb_count_key(struct afl_state* afl, int bb_random_val) {
    struct basic_block_count *bbc = NULL;
    HASH_FIND_INT(afl->bb2count, &bb_random_val, bbc);
    
    // if bbc not found in bb2count, create a new one
    if (bbc == NULL) {
        bbc = (struct basic_block_count*)malloc(sizeof(struct basic_block_count));
        bbc->bb_random = bb_random_val;
        bbc->count = 0;
        // add bbc->bb_random as key bbc as value to bb2count hashtable
        HASH_ADD_INT(afl->bb2count, bb_random, bbc);
    }
}
...
/* ----- afl-fuzz-json.c ----- */
```



## intelliSense conflicting

vscode 安装许多插件, 可能会冲突, 比如 `clangd` 和 `cpptools` 的 intelliSense 会冲突, 

解决: disable (workspace) 其中之一

