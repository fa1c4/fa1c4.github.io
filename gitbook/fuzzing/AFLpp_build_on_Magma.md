# AFL++ build for Magma

AFL++ <sup>[1]</sup> 是AFL的增强版, 集成了各种fuzzing优化方案/技术, 在工业界通常作为一个重要的模糊测试工具, 在学术界则作为重要的Baseline. 并且AFL++是一直在维护和更新的, 所以也可以当做挖漏洞的标配Fuzzer.

主要包括基于原版AFL改进的的几种优化版本:

+ **AFLFast** <sup>[3]</sup>: 给触发低频路径的种子赋予更高能量
+ **MOpt** <sup>[4]</sup>: 自定义粒子群优化 (Particle Swarm Optimization, PSO) 算法, 根据测试效果寻找算子的最优选择概率分布

+ **LAF-Intel** <sup>[5]</sup>: LLVM pass反优化条件分支提高AFL进入高难度条件分支的概率

+ **RedQueen** <sup>[6]</sup>: 定义 Input-To-State (I2S) 自动解决 magic byte 和 checksum 检查

+ **AFLSmart** <sup>[7]</sup>: 人工配置结构化输入用例, 测试时进行结构化变异

[Magma](https://hexhive.epfl.ch/magma/) <sup>[2]</sup> 是Greybox Fuzzing论文中重要的Benchmark之一. 

所以本文主要介绍AFL++ (原版) 在Magma benchmark上的安装, 配置, 使用的流程.





## AFL++ 配置

OS: Ubuntu20.04 + | Dependencies: python-3.10+, llvm-19, gcc-9.4.0

安装环境依赖 (非私有环境下按需安装, 避免破坏已有环境)

```shell
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install -y build-essential python3-dev automake cmake git flex bison libglib2.0-dev libpixman-1-dev python3-setuptools
sudo apt-get install -y lld llvm llvm-dev clang
sudo apt-get install -y gcc-$(gcc --version|head -n1|sed 's/.* //'|sed 's/\..*//')-plugin-dev libstdc++-$(gcc --version|head -n1|sed 's/.* //'|sed 's/\..*//')-dev
sudo apt-get install -y ninja-build # for QEMU mode
```

下载和编译AFL++

```shell
# cd to home path
git clone https://github.com/AFLplusplus/AFLplusplus.git
cd AFLplusplus
make clean # if run again
make distrib # for binary level and source code level
# make source-only # for only source code level 
sudo make install # not necessary
```

报错解决

1. 缺少 libpython3.xx.so.1.0: `sudo apt-get install libpython3.xx-dev`
2. 缺少 `gcc-X-plugin-dev` 头文件: `sudo apt-get install gcc-xx-plugin-dev` 或者 见上面依赖环境安装命令





## Magma 配置

Magma官方文档写明白了如何编译和使用测试集, 但是默认都为 Docker 环境下的编译和测试. 对于科研工作有时不太方便, 而且有的fuzzer不兼容, 所以这里给出物理机环境下的测试集程序编译和测试的流程.

```shell
# install dependencies
apt-get update && apt-get install -y util-linux inotify-tools docker.io

# clone magma
git clone --branch v1.2.1 https://github.com/HexHive/magma.git
```



编译AFL driver

```shell
export CC=/path/to/AFLplusplus/afl-clang-fast
export CXX=/path/to/AFLplusplus/afl-clang-fast++

cd magma/fuzzer/afl/src
$CXX -std=c++11 -c "afl_driver.cpp" -fPIC -o "./afl_driver.o"
```

报错解决

1. use of undeclared identifier 'va_start' and 'va_end': `afl_driver.cpp` 添加 `#include <cstdarg>`



下载编译目标程序

#### libpng

```shell
# patch source code
cd magma/
git clone --no-checkout https://hub.fastgit.org/glennrp/libpng.git targets/libpng/repo
git -C "targets/libpng/repo" checkout a37d4836519517bdce6cb9d956092321eca3e73b
./magma/apply_patches.sh # not neccessary for running

# make target lib
cd targets/libpng/repo
autoreconf -f -i
./configure --with-libpng-prefix=MAGMA_ --disable-shared
make -j$(nproc) clean
make -j$(nproc) libpng16.la
cp .libs/libpng16.a ./

# compile target binary !modify the /path/to/.../afl_driver.o
$CXX -std=c++11 -I. contrib/oss-fuzz/libpng_read_fuzzer.cc -o ./libpng_read_fuzzer ./libpng16.a /path/to/magma/fuzzers/afl/src/afl_driver.o -lstdc++ -lz
```

报错解决

1. use of undeclared identifier 'malloc' and free': `libpng_read_fuzzer.cc` add `#include <cstdlib>`
2. *之后同理



#### libtiff

```shell
# patch source code
cd magma/
git clone --no-checkout https://gitlab.com/libtiff/libtiff.git targets/libtiff/repo
git -C "targets/libtiff/repo" checkout c145a6c14978f73bb484c955eb9f84203efcb12e
cp targets/libtiff/src/tiff_read_rgba_fuzzer.cc targets/libtiff/repo/contrib/oss-fuzz/tiff_read_rgba_fuzzer.cc
./magma/apply_patches.sh # not neccessary for running

# compile tiffcp
cd targets/libtiff/repo
mkdir work
./autogen.sh
./configure --disable-shared --prefix=`pwd`/work
make -j$(nproc) clean
make -j$(nproc)
make install

# compile tiff_read_rgba_fuzzer
$CXX -std=c++11 -I`pwd`/work/include contrib/oss-fuzz/tiff_read_rgba_fuzzer.cc -o ./tiff_read_rgba_fuzzer `pwd`/work/lib/libtiffxx.a `pwd`/work/lib/libtiff.a -lz -ljpeg -ljbig -Wl,-Bstatic -llzma -Wl,-Bdynamic /path/to/magma/fuzzers/src/afl_driver.o -lstdc++
```

报错解决

1. /usr/bin/ld: cannot find -ljbig: `sudo apt install libjbig-dev`



#### libxml2

```shell
# patch source code
cd magma/
git clone --no-checkout https://gitlab.gnome.org/GNOME/libxml2.git targets/libxml2/repo
git -C "targets/libxml2/repo" checkout ec6e3efb06d7b15cf5a2328fabd3845acea4c815
./magma/apply_patches.sh # not neccessary for running

# compile xmllint
cd targets/libxml2/repo
./autogen.sh \
	--with-http=no \
	--with-python=no \
	--with-lzma=yes \
	--with-threads=no \
	--disable-shared
make -j$(nproc) clean
make -j$(nproc) all

# compile libxml2_xml_read_memory_fuzzer & libxml2_xml_reader_for_file_fuzzer
for fuzzer in libxml2_xml_read_memory_fuzzer libxml2_xml_reader_for_file_fuzzer; do \
$CXX -std=c++11 -Iinclude/ -I"`pwd`/../src/" "`pwd`/../src/$fuzzer.cc" -o "./$fuzzer" .libs/libxml2.a /path/to/magma/fuzzers/afl/src/afl_driver.o -lstdc++ -lz -llzma; done;
```



#### openssl

```shell
# patch source code
cd magma/
git clone --no-checkout https://github.com/openssl/openssl.git targets/openssl/repo
git -C "targets/openssl/repo" checkout 3bd5319b5d0df9ecf05c8baba2c401ad8e3ba130
cp targets/openssl/src/abilist.txt targets/openssl/repo/abilist.txt
./magma/apply_patches.sh # not neccessary for running

# set environment variables
cd targets/openssl/repo
CONFIGURE_FLAGS=""
if [[ $CFLAGS = *sanitize=memory* ]]
then
  CONFIGURE_FLAGS="no-asm"
fi
export LDLIBS="/path/to/magma/fuzzers/afl/src/afl_driver.o -lstdc++"

# compile target binary
./config --debug enable-fuzz-libfuzzer enable-fuzz-afl disable-tests -DPEDANTIC \
    -DFUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION no-shared no-module \
    enable-tls1_3 enable-rc5 enable-md2 enable-ec_nistp_64_gcc_128 enable-ssl3 \
    enable-ssl3-method enable-nextprotoneg enable-weak-ssl-ciphers \
    -fno-sanitize=alignment $CONFIGURE_FLAGS
make -j$(nproc) clean
make -j$(nproc) # buggy for now
```



#### php

```shell
# patch source code
cd magma/
git clone --no-checkout https://github.com/php/php-src.git targets/php/repo
git -C "targets/php/repo" checkout bc39abe8c3c492e29bc5d60ca58442040bbf063b
git clone --no-checkout https://github.com/kkos/oniguruma.git targets/php/repo/oniguruma
git -C "targets/php/repo/oniguruma" checkout 227ec0bd690207812793c09ad70024707c405376
./magma/apply_patches.sh # not neccessary for running

# set environment variables
export ONIG_CFLAGS="-I`pwd`/oniguruma/src"
export ONIG_LIBS="-L`pwd`/oniguruma/src/.libs -lonig"
export EXTRA_CFLAGS="-fno-sanitize=object-size"
export EXTRA_CXXFLAGS="-fno-sanitize=object-size"

# compile oniguruma
./buildconf
LIB_FUZZING_ENGINE="-Wall" CFLAGS="-DXXH_VECTOR=0 -fno-slp-vectorize" ./configure \
    --disable-all \
    --enable-option-checking=fatal \
    --enable-fuzzer \
    --enable-exif \
    --enable-phar \
    --enable-intl \
    --enable-mbstring \
    --without-pcre-jit \
    --disable-phpdbg \
    --disable-cgi \
    --with-pic
make clean

pushd oniguruma
autoreconf -vfi
./configure --disable-shared
make -j$(nproc)
popd

# compile php
# modify Makefile
# FUZZING_LIB = -Wall # modify to:
# FUZZING_LIB = -Wall xxxpath/afl_driver.o -lstdc++
make -j$(nproc)
```



#### poppler

```shell
# patch source code
cd magma/
git clone --no-checkout https://gitlab.com/libtiff/libtiff.git targets/poppler/repo
git -C "targets/poppler/repo" checkout c145a6c14978f73bb484c955eb9f84203efcb12e
git clone --no-checkout git://git.sv.nongnu.org/freetype/freetype2.git targets/poppler/repo/freetype2
git -C "targets/poppler/repo/freetype2" checkout 50d0033f7ee600c5f5831b28877353769d1f7d48
./magma/apply_patches.sh # not neccessary for running

# compile freetype2
cd targets/poppler/repo/freetype2
./autogen.sh
./configure --prefix=`pwd`/../work --disable-shared PKG_CONFIG_PATH="`pwd`/../work/lib/pkgconfig"
make -j$(nproc) clean
make -j$(nproc)
make install

# compile pdfimages & pdftoppm
cd work
mkdir poppler
cd poppler
EXTRA=""
test -n "$AR" && EXTRA="$EXTRA -DCMAKE_AR=$AR"
test -n "$RANLIB" && EXTRA="$EXTRA -DCMAKE_RANLIB=$RANLIB"
cmake ../../ $EXTRA -DCMAKE_BUILD_TYPE=debug -DBUILD_SHARED_LIBS=OFF -DFONT_CONFIGURATION=generic -DBUILD_GTK_TESTS=OFF -DBUILD_QT5_TESTS=OFF -DBUILD_CPP_TESTS=OFF -DENABLE_LIBPNG=ON -DENABLE_LIBTIFF=ON -DENABLE_LIBJPEG=ON -DENABLE_SPLASH=ON -DENABLE_UTILS=ON -DWITH_Cairo=ON -DENABLE_CMS=none -DENABLE_LIBCURL=OFF -DENABLE_GLIB=OFF -DENABLE_GOBJECT_INTROSPECTION=OFF -DENABLE_QT5=OFF -DENABLE_LIBCURL=OFF -DWITH_NSS3=OFF \
-DFREETYPE_INCLUDE_DIRS="`pwd`/../include/freetype2" \
-DFREETYPE_LIBRARY="`pwd`/../lib/libfreetype.a" \
-DICONV_LIBRARIES="/usr/lib/x86_64-linux-gnu/libc.so" \
-DCMAKE_EXE_LINKER_FLAGS_INIT="/path/to/magma/fuzzers/afl/src/afl_driver.o -lstdc++"
EXTRA=""

# compile pdf_fuzzer
$CXX -std=c++11 -I"`pwd`/../poppler/cpp" -I"`pwd`/../../cpp" "`pwd`/../../../src/pdf_fuzzer.cc" -o "./pdf_fuzzer" "`pwd`/../poppler/cpp/libpoppler-cpp.a" "`pwd`/../poppler/libpoppler.a" "`pwd`/../lib/libfreetype.a" /path/to/magma/fuzzers/afl/src/afl_driver.o -lstdc++ -ljpeg -lz -lopenjp2 -lpng -ltiff -llcms2 -lm -lpthread -pthread
```



#### sqlite3

```shell
# patch source code
cd magma/
curl "https://www.sqlite.org/src/tarball/sqlite.tar.gz?r=8c432642572c8c4b" \
  -o "targets/sqlite3/sqlite.tar.gz" && \
mkdir -p "targets/sqlite3/repo" && \
tar -C "targets/sqlite3/repo" --strip-components=1 -xzf "targets/sqlite3/sqlite.tar.gz"
./magma/apply_patches.sh # not neccessary for running

# set environment variables
export CFLAGS="-DSQLITE_MAX_LENGTH=128000000 \
               -DSQLITE_MAX_SQL_LENGTH=128000000 \
               -DSQLITE_MAX_MEMORY=25000000 \
               -DSQLITE_PRINTF_PRECISION_LIMIT=1048576 \
               -DSQLITE_DEBUG=1 \
               -DSQLITE_MAX_PAGE_COUNT=16384"
               
# compile sqlite3
cd targets/sqlite3/repo
../configure --disable-shared --enable-rtree
make clean
make -j$(nproc)
make sqlite3.c

# compile sqlite3_fuzz
$CC -I. "`pwd`/../test/ossfuzz.c" "./sqlite3.o" -o "./sqlite3_fuzz" /path/to/magma/fuzzers/afl/src/afl_driver.o -lstdc++ -pthread -ldl -lm
```





## AFL++ running on Magma

before running the fuzzer

```shell
export AFL_SKIP_CPUFREQ=1
```

#### libpng

```shell
# make sure the directories exist
make -p /path/to/fuzz_out/libpng

# start fuzzing
/path/to/AFLplusplus/afl-fuzz -i /path/to/magma/targets/libpng/corpus/libpng_read_fuzzer -o /path/to/fuzz_out/libpng -- /path/to/magma/targets/libpng/repo/libpng_read_fuzzer
```



#### libtiff

```shell
# make sure the directories exist
make -p /path/to/fuzz_out/libtiff

# start fuzzing
/path/to/AFLplusplus/afl-fuzz -i /path/to/magma/targets/libtiff/corpus/corpus/libtiff -o /path/to/fuzz_out/libtiff -- /path/to/magma/targets/libtiff/repo/tiff_read_rgba_fuzzer
```



#### libxml2

```shell
# make sure the directories exist
make -p /path/to/fuzz_out/libxml2_file
make -p /path/to/fuzz_out/libxml2_memory

# start fuzzing libxml2_xml_reader_for_file_fuzzer
/path/to/AFLplusplus/afl-fuzz -i /path/to/magma/targets/libxml2/corpus/libxml2_xml_reader_for_file_fuzzer -o /path/to/fuzz_out/libxml2_file -- /path/to/magma/targets/libxml2/repo/libxml2_xml_reader_for_file_fuzzer

# start fuzzing libxml2_xml_read_memory_fuzzer
/path/to/AFLplusplus/afl-fuzz -i /path/to/magma/targets/libxml2/corpus/libxml2_xml_read_memory_fuzzer -o /path/to/fuzz_out/libxml2_memory -- /path/to/magma/targets/libxml2/repo/libxml2_xml_read_memory_fuzzer
```



#### openssl

```shell
# make sure the directories exist
make -p /path/to/fuzz_out/asn1parse

# start fuzzing
/path/to/AFLplusplus/afl-fuzz -i /path/to/magma/targets/openssl/corpus/asn1parse -o /path/to/fuzz_out/asn1parse -- /path/to/magma/targets/openssl/repo/asn1parse
```



#### php

```shell
# make sure the directories exist
make -p /path/to/fuzz_out/php_xxx

# start fuzzing
/path/to/AFLplusplus/afl-fuzz -i /path/to/magma/targets/php/corpus/php_fuzz_xxx -o /path/to/fuzz_out/php_xxx -- /path/to/magma/targets/php/repo/php_fuzz_xxx
```



#### poppler

```shell
# make sure the directories exist
make -p /path/to/fuzz_out/poppler

# start fuzzing
/path/to/AFLplusplus/afl-fuzz -i /path/to/magma/targets/poppler/corpus/pdf_fuzzer -o /path/to/fuzz_out/poppler -- /path/to/magma/targets/poppler/repo/pdf_fuzzer
```



#### sqlite3

```shell
# make sure the directories exist
make -p /path/to/fuzz_out/sqlite3

# start fuzzing
/path/to/AFLplusplus/afl-fuzz -i /path/to/magma/targets/sqlite3/corpus/sqlite3_fuzz -o /path/to/fuzz_out/sqlite3 -- /path/to/magma/targets/sqlite3/repo/work/sqlite3_fuzz
```





## Summary

AFL++ 在 Magma 上运行的方式, 类似也可以应用到其他 fuzzer 上.





## References

[1] https://github.com/AFLplusplus/AFLplusplus

[2] Hazimeh, Ahmad, Adrian Herrera, and Mathias Payer. "Magma: A ground-truth fuzzing benchmark." *Proceedings of the ACM on Measurement and Analysis of Computing Systems* 4.3 (2020): 1-29.

[3] https://github.com/BT123/aflfast

[4] https://www.usenix.org/conference/usenixsecurity19/presentation/lyu

[5] https://lafintel.wordpress.com/

[6] https://www.ndss-symposium.org/ndss-paper/redqueen-fuzzing-with-input-to-state-correspondence/

[7] https://github.com/aflsmart/aflsmart

[8] https://zhuanlan.zhihu.com/p/430280845

