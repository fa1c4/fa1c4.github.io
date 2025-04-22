# Add New Test in AFL++ Framework

AFL++ 框架代码量非常庞大, C/C++ LoC >= 170w. 

在这个框架下增加代码必须要经过模块测试, 否则新增代码的正确性存疑.

本文记录在 AFL++ 下对某一模块增加测试的流程.



## Create Test

build targets 包含如下选项

- all: the main AFL++ binaries and llvm/gcc instrumentation
- binary-only: everything for binary-only fuzzing: frida_mode, nyx_mode, qemu_mode, frida_mode, unicorn_mode, coresight_mode, libdislocator, libtokencap
- source-only: everything for source code fuzzing: nyx_mode, libdislocator, libtokencap
- distrib: everything (for both binary-only and source code fuzzing)
- man: creates simple man pages from the help option of the programs
- install: installs everything you have compiled with the build options above
- clean: cleans everything compiled, not downloads (unless not on a checkout)
- deepclean: cleans everything including downloads
- code-format: format the code, do this before you commit and send a PR please!
- tests: runs test cases to ensure that all features are still working as they should
- unit: perform unit tests (based on cmocka)
- help: shows these build options



和测试功能相关的选项为 `tests`, 正常编译后测试 `make tests` 会显示 `afl-showmap, afl-fuzz, afl-cmin, afl-tmin` 每个重要组件是否按预期工作.



打开查看 `tests` 相关编译逻辑, 调用链: `Makefile -> GNUmakefile -> test/test-all.sh -> test/test-pre.sh -> test/test-basic.sh`, 原 shell 代码很丑陋, 经过人工重排格式, 增加可读性. 对逻辑的分析已注释在代码之间.

```sh
#!/bin/sh

. ./test-pre.sh

OS=$(uname -s)

AFL_COMPILER=afl-clang-fast # define compiler as afl-clang-fast

$ECHO "$BLUE[*] Testing: ${AFL_COMPILER}, afl-showmap, afl-fuzz, afl-cmin and afl-tmin"
 
 # check if compiler afl-clang-fast exists, check if afl-showmap, afl-fuzz exists
 # if all exist then goto failure at the end
 test -e ../${AFL_COMPILER} -a -e ../afl-showmap -a -e ../afl-fuzz && 
 { # checking existence pass
 
   # compile test-instr.c with afl-clang-fast to test-instr.plain
   ../${AFL_COMPILER} -o test-instr.plain -O0 ../test-instr.c > /dev/null 2>&1
   
   # compile test-compcov.c with afl-clang-fast to test-compcov.harden
   AFL_HARDEN=1 ../${AFL_COMPILER} -o test-compcov.harden test-compcov.c > /dev/null 2>&1
   
   # check if compiled test-instr.plain exists 
   test -e test-instr.plain && 
   {
   
    # print comilation succeeded
    $ECHO "$GREEN[+] ${AFL_COMPILER} compilation succeeded"
    
    # run afl-showmap to instrument test-instr.plain
    echo 0 | AFL_QUIET=1 ../afl-showmap -m ${MEM_LIMIT} -o test-instr.plain.0 -r -- ./test-instr.plain > /dev/null 2>&1
    
    # run afl-showmap to instrument test-instr.plain again
    AFL_QUIET=1 ../afl-showmap -m ${MEM_LIMIT} -o test-instr.plain.1 -r -- ./test-instr.plain < /dev/null > /dev/null 2>&1
    
    # check if instrumentation succeed
    test -e test-instr.plain.0 -a -e test-instr.plain.1 && 
    {
      # compare test-instr.plain.0 with test-instr.plain.1 to check if instrumentation logic work as expectation
      diff test-instr.plain.0 test-instr.plain.1 > /dev/null 2>&1 && 
      {
        # different input should result in different instrumentation
        $ECHO "$RED[!] ${AFL_COMPILER} instrumentation should be different on different input but is not"
        CODE=1 
      } || 
      {
        # instrumentation work correctly regard to different input leading to different results
        $ECHO "$GREEN[+] ${AFL_COMPILER} instrumentation present and working correctly"
      }
    } || # checked if instrumentation succeed
    { # instrumentation failed
      $ECHO "$RED[!] ${AFL_COMPILER} instrumentation failed"
      CODE=1
    }
    # remove instrumented files
    rm -f test-instr.plain.0 test-instr.plain.1
    
    SKIP=
    # TUPLES store instrumented locations information
    TUPLES=`echo 1|AFL_QUIET=1 ../afl-showmap -m ${MEM_LIMIT} -o /dev/null -- ./test-instr.plain 2>&1 | grep Captur | awk '{print$3}'`
    
    # check if the count of instrumentation is correct 
    test "$TUPLES" -gt 1 -a "$TUPLES" -lt 22 && 
    {
      $ECHO "$GREEN[+] ${AFL_COMPILER} run reported $TUPLES instrumented locations which is fine"
    } || 
    {
      $ECHO "$RED[!] ${AFL_COMPILER} instrumentation produces weird numbers: $TUPLES"
      CODE=1
    }
    
    # if instrumented locations less than 3, skip the remaining checking
    test "$TUPLES" -lt 3 && SKIP=1
    
    true  # this is needed because of the test above
   } || # checked if instrumentation succeed
   { # instrumentation failed
    $ECHO "$RED[!] ${AFL_COMPILER} failed"
    echo CUT------------------------------------------------------------------CUT
    uname -a
    ../${AFL_COMPILER} -o test-instr.plain -O0 ../test-instr.c
    echo CUT------------------------------------------------------------------CUT
    CODE=1
   }
   
   # check if test-compcov.harden exists
   test -e test-compcov.harden && 
   {
    # nm to check if stack protection enabled in test-compcov.harden
    nm test-compcov.harden | grep -Eq 'stack_chk_fail|fstack-protector-all|fortified' > /dev/null 2>&1 && 
    {
      $ECHO "$GREEN[+] ${AFL_COMPILER} hardened mode succeeded and is working"
    } || 
    { # harden mode not passed
      $ECHO "$RED[!] ${AFL_COMPILER} hardened mode is not hardened"
      env | grep -E 'AFL|PATH|LLVM'
      AFL_DEBUG=1 AFL_HARDEN=1 ../${AFL_COMPILER} -o test-compcov.harden test-compcov.c
      nm test-compcov.harden
      CODE=1
    }
    rm -f test-compcov.harden
   } || # checked if test-compcov.harden exists
   { # test-compcov.harden not exists
    $ECHO "$RED[!] ${AFL_COMPILER} hardened mode compilation failed"
    CODE=1
   }
   
   # now we want to be sure that afl-fuzz is working
   
   # make sure crash reporter is disabled on Mac OS X
   (test "$OS" = "Darwin" && test $(launchctl list 2>/dev/null | grep -q '\.ReportCrash$') && {
    $ECHO "$RED[!] we cannot run afl-fuzz with enabled crash reporter. Run 'sudo sh afl-system-config'.$RESET"
    true
   }) || # disable crash reporter if on Mac OS X
   { # main logic of checking afl-fuzz & afl-*min
   
    mkdir -p in # create 'in' directory
    echo 0 > in/in # create 'in/in' file and write 0 into it
    
    # if SKIP is empty then continue checking afl-fuzz running, otherwise skip checking
    test -z "$SKIP" && 
    {
      $ECHO "$GREY[*] running afl-fuzz for ${AFL_COMPILER}, this will take approx 10 seconds"
      {
        # fuzzing test-instr.plain
        ../afl-fuzz -V07 -m ${MEM_LIMIT} -i in -o out -- ./test-instr.plain >>errors 2>&1
      } >>errors 2>&1
      
      # check if afl-fuzz generated out queue
      test -n "$( ls out/default/queue/id:000002* 2>/dev/null )" && 
      {
        # queue generated means that afl-fuzz work correctly 
        $ECHO "$GREEN[+] afl-fuzz is working correctly with ${AFL_COMPILER}"
      } || { # otherwise, afl-fuzz has errors
        echo CUT------------------------------------------------------------------CUT
        cat errors
        echo CUT------------------------------------------------------------------CUT
        $ECHO "$RED[!] afl-fuzz is not working correctly with ${AFL_COMPILER}"
        CODE=1
      }
    } # SKIP checked
    
    # create 'in/in2' and 'in/in3' files
    echo 000000000000000000000000 > in/in2
    echo 111 > in/in3
    
    # in macOS system, afl-cmin doesn't work, so skip it.
    test "$OS" = "Darwin" && 
    {
      $ECHO "$GREY[*] afl-cmin not available on macOS, cannot test afl-cmin"
    } || 
    { # check afl-cmin 
      
      # create in2 directory to store afl-cmin results
      mkdir -p in2
      # afl-cmin to minimize corpus
      ../afl-cmin -m ${MEM_LIMIT} -i in -o in2 -- ./test-instr.plain >/dev/null 2>&1 # why is afl-forkserver writing to stderr?
      
      # get the minimized count of corpus CNT
      CNT=`ls in2/* 2>/dev/null | wc -l`
      # check if $CNT = 2, which is correct minimized number of testcases, otherwise is wrong 
      case "$CNT" in
        *2) $ECHO "$GREEN[+] afl-cmin correctly minimized the number of testcases" ;;
        *)  $ECHO "$RED[!] afl-cmin did not correctly minimize the number of testcases ($CNT)"
            CODE=1
            ;;
      esac
      rm -f in2/in*
    } # checked afl-cmin
    
    # check another afl-cmin bash script, same logic as above afl-cmin
    export AFL_QUIET=1
    # check if bash is available
    if command -v bash >/dev/null ; then 
    {
      ../afl-cmin.bash -m ${MEM_LIMIT} -i in -o in2 -- ./test-instr.plain >/dev/null
      CNT=`ls in2/* 2>/dev/null | wc -l`
      case "$CNT" in
        *2) $ECHO "$GREEN[+] afl-cmin.bash correctly minimized the number of testcases" ;;
        *)  $ECHO "$RED[!] afl-cmin.bash did not correctly minimize the number of testcases ($CNT)"
            CODE=1
            ;;
        esac
    } else 
    {
      $ECHO "$GREY[*] no bash available, cannot test afl-cmin.bash"
    }
    fi
    
    # check afl-tmin, similar with afl-cmin
    ../afl-tmin -m ${MEM_LIMIT} -i in/in2 -o in2/in2 -- ./test-instr.plain > /dev/null 2>&1
    SIZE=`ls -l in2/in2 2>/dev/null | awk '{print$5}'`
    test "$SIZE" = 1 && $ECHO "$GREEN[+] afl-tmin correctly minimized the testcase"
    test "$SIZE" = 1 || 
    {
       $ECHO "$RED[!] afl-tmin did incorrectly minimize the testcase to $SIZE"
       CODE=1
    }
    rm -rf in out errors in2
    unset AFL_QUIET
   } # end of check afl-fuzz & afl-*min
   
   rm -f test-instr.plain
   
 } # afl-clang-fast, afl-showmap, afl-fuzz all exist checked 
 || 
 { # failure at the end
   $ECHO "$YELLOW[-] afl is not compiled, cannot test"
   INCOMPLETE=1
 }

# continue post test
. ./test-post.sh
```



看懂 `make tests` 逻辑之后, 要添加自定义的测试模块就很自然了, 主要包括 `test_your_module.sh` 和 `test_your_module.c`, 把测试所需要运行的命令行写入前者 `.sh` 脚本中, 把测试的驱动逻辑写到后者 `.c` 文件中. 

整个测试之前, 需要编译通过 `afl-showmap, afl-fuzz, afl-*min` 等程序. 所以加入自定义模块的测试, 也需要自行编译通过相关驱动模块程序.



## Run Test

比如测试一个框架无关代码: 在 `test` 目录下创建测试输出 `hello AFL++` 的驱动程序

`test_hello_aflpp.sh`

```sh
#!/bin/bash

# define variables
AFL_COMPILER=../afl-clang-fast
SOURCE_FILE=test_hello_aflpp.c
OUTPUT_FILE=test_hello_aflpp
EXPECTED_OUTPUT="hello AFL++"

# compile test program
echo "[*] Compiling $SOURCE_FILE with $AFL_COMPILER..."
$AFL_COMPILER -o $OUTPUT_FILE $SOURCE_FILE

# check if compilation succeed
if [ ! -f $OUTPUT_FILE ]; then
  echo "[!] Compilation failed."
  exit 1
fi

# execute program and get output
echo "[*] Running $OUTPUT_FILE..."
OUTPUT=$(./$OUTPUT_FILE)

# test program outputs as expectation
if [ "$OUTPUT" == "$EXPECTED_OUTPUT" ]; then
  echo "[+] Test passed! Output is correct."
else
  echo "[!] Test failed! Output is incorrect."
  echo "Expected: $EXPECTED_OUTPUT"
  echo "Got: $OUTPUT"
  exit 1
fi
```

`test_hello_aflpp.c`

```c
#include <stdio.h>

int main() {
    printf("hello AFL++\n");
    return 0;
}
```



执行 `test_hello_aflpp.sh` 运行结果 (当然可以使用其他编译器进行测试)

```shell
[*] Compiling test_hello_aflpp.c with ../afl-clang-fast...
afl-cc++4.31c by Michal Zalewski, Laszlo Szekeres, Marc Heuse - mode: LLVM-PCGUARD
SanitizerCoveragePCGUARD++4.31c
[+] Instrumented 1 locations with no collisions (non-hardened mode) of which are 0 handled and 0 unhandled selects.
[*] Running test_hello_aflpp...
[+] Test passed! Output is correct.
```



测试框架相关代码类似, 可在 `test` 目录创建一个 `.c` 文件包含目标头文件, 并使用目标函数完成实际操作并输出结果, 然后单独编译一个可执行文件. 在 `.sh` 中提供目标测试例的正确结果, 再运行测试对比运行是否符合预期. 





## Reference

[1] https://aflplus.plus/docs/install/

[2] https://www.geeksforgeeks.org/shell-scripting-test-command/