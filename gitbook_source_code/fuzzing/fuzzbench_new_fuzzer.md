# Add New Fuzzer for Fuzzbench

在 Fuzzbench 中增加新的 fuzzer 适配实验环境. Refer to [https://google.github.io/fuzzbench/getting-started/adding-a-new-fuzzer/](https://google.github.io/fuzzbench/getting-started/adding-a-new-fuzzer/)

直接增加的 Fuzzer 条件: 1.不依赖 IDApro 的静态分析环境, 或者预处理流程能够在 ubuntu20.04 环境下完成, 2.动态调整流程可以在同一个 fuzzing docker 内进行.



## Create fuzzer directory

在 `fuzzbench/fuzzers` 目录下创建文件夹 (只能是小写字母和 _ 组成), 用来存放新增的 fuzzer 文件

```shell
cd fuzzbench/fuzzers
mkdir new_fuzzer_name

# test if the name valid
# cd to fuzzbench root directory
make presubmit
```

确保新增的 fuzzer_name 能通过检查, 否则按报错信息排查解决



## Create fuzzer files

在已有的 `fuzzers` 目录下查看各个 fuzzer 目录文件, 主要包括三个文件

```shell
builder.Dockerfile  fuzzer.py  runner.Dockerfile
```

首先是 `builder.Dockerfile` 

```dockerfile
ARG parent_image						   # being set outside docker
FROM $parent_image                         # Base builder image (Ubuntu 20.04, with latest Clang).

RUN apt-get update && \                    # Install any system dependencies to build your fuzzer.
    apt-get install -y pkg1 pkg2 ...

RUN git clone <git_url> /fuzzer_src        # Clone your fuzzer's sources.

RUN cd /fuzzer_src && make                 # Build your fuzzer using its preferred build system.

# Build your `FUZZER_LIB`.
# See section below on "What is `FUZZER_LIB`?" for more details.
RUN git clone <git_url> /fuzzer_lib_src

RUN cd /fuzzer_lib_src && clang++ fuzzer_lib.o
```

然后是 `runner.Dockerfile`

```dockerfile
FROM gcr.io/fuzzbench/base-image           # Base image (Ubuntu 20.04).

RUN apt-get update && \                    # Install any runtime dependencies for your fuzzer.
    apt-get install -y pkg1 pkg2 ...
```

最后是 `fuzzer.py`, 一般基于 `afl` 修改代码的 `aflxx` 版本, 可以直接沿用 `fuzzers/afl` 目录下的 `fuzzer.py`

```python
"""Integration code for AFL fuzzer."""

import json
import os
import shutil
import subprocess

from fuzzers import utils


def prepare_build_environment():
    """Set environment variables used to build targets for AFL-based
    fuzzers."""
    cflags = ['-fsanitize-coverage=trace-pc-guard']
    utils.append_flags('CFLAGS', cflags)
    utils.append_flags('CXXFLAGS', cflags)

    os.environ['CC'] = 'clang'
    os.environ['CXX'] = 'clang++'
    os.environ['FUZZER_LIB'] = '/libAFL.a'


def build():
    """Build benchmark."""
    prepare_build_environment()

    utils.build_benchmark()

    print('[post_build] Copying afl-fuzz to $OUT directory')
    # Copy out the afl-fuzz binary as a build artifact.
    shutil.copy('/afl/afl-fuzz', os.environ['OUT'])


def get_stats(output_corpus, fuzzer_log):  # pylint: disable=unused-argument
    """Gets fuzzer stats for AFL."""
    # Get a dictionary containing the stats AFL reports.
    stats_file = os.path.join(output_corpus, 'fuzzer_stats')
    if not os.path.exists(stats_file):
        print('Can\'t find fuzzer_stats')
        return '{}'
    with open(stats_file, encoding='utf-8') as file_handle:
        stats_file_lines = file_handle.read().splitlines()
    stats_file_dict = {}
    for stats_line in stats_file_lines:
        key, value = stats_line.split(': ')
        stats_file_dict[key.strip()] = value.strip()

    # Report to FuzzBench the stats it accepts.
    stats = {'execs_per_sec': float(stats_file_dict['execs_per_sec'])}
    return json.dumps(stats)


def prepare_fuzz_environment(input_corpus):
    """Prepare to fuzz with AFL or another AFL-based fuzzer."""
    # Tell AFL to not use its terminal UI so we get usable logs.
    os.environ['AFL_NO_UI'] = '1'
    # Skip AFL's CPU frequency check (fails on Docker).
    os.environ['AFL_SKIP_CPUFREQ'] = '1'
    # No need to bind affinity to one core, Docker enforces 1 core usage.
    os.environ['AFL_NO_AFFINITY'] = '1'
    # AFL will abort on startup if the core pattern sends notifications to
    # external programs. We don't care about this.
    os.environ['AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES'] = '1'
    # Don't exit when crashes are found. This can happen when corpus from
    # OSS-Fuzz is used.
    os.environ['AFL_SKIP_CRASHES'] = '1'
    # Shuffle the queue
    os.environ['AFL_SHUFFLE_QUEUE'] = '1'

    # AFL needs at least one non-empty seed to start.
    utils.create_seed_file_for_empty_corpus(input_corpus)


def check_skip_det_compatible(additional_flags):
    """ Checks if additional flags are compatible with '-d' option"""
    # AFL refuses to take in '-d' with '-M' or '-S' options for parallel mode.
    # (cf. https://github.com/google/AFL/blob/8da80951/afl-fuzz.c#L7477)
    if '-M' in additional_flags or '-S' in additional_flags:
        return False
    return True


def run_afl_fuzz(input_corpus,
                 output_corpus,
                 target_binary,
                 additional_flags=None,
                 hide_output=False):
    """Run afl-fuzz."""
    # Spawn the afl fuzzing process.
    print('[run_afl_fuzz] Running target with afl-fuzz')
    command = [
        './afl-fuzz',
        '-i',
        input_corpus,
        '-o',
        output_corpus,
        # Use no memory limit as ASAN doesn't play nicely with one.
        '-m',
        'none',
        '-t',
        '1000+',  # Use same default 1 sec timeout, but add '+' to skip hangs.
    ]
    # Use '-d' to skip deterministic mode, as long as it it compatible with
    # additional flags.
    if not additional_flags or check_skip_det_compatible(additional_flags):
        command.append('-d')
    if additional_flags:
        command.extend(additional_flags)
    dictionary_path = utils.get_dictionary_path(target_binary)
    if dictionary_path:
        command.extend(['-x', dictionary_path])
    command += [
        '--',
        target_binary,
        # Pass INT_MAX to afl the maximize the number of persistent loops it
        # performs.
        '2147483647'
    ]
    print('[run_afl_fuzz] Running command: ' + ' '.join(command))
    output_stream = subprocess.DEVNULL if hide_output else None
    subprocess.check_call(command, stdout=output_stream, stderr=output_stream)


def fuzz(input_corpus, output_corpus, target_binary):
    """Run afl-fuzz on target."""
    prepare_fuzz_environment(input_corpus)

    run_afl_fuzz(input_corpus, output_corpus, target_binary)
```



## Testing

```shell
export FUZZER_NAME=afl
export BENCHMARK_NAME=libpng-1.2.56
make build-$FUZZER_NAME-$BENCHMARK_NAME
# make debug-builder-$FUZZER_NAME-$BENCHMARK_NAME

make run-$FUZZER_NAME-$BENCHMARK_NAME
# make test-run-$FUZZER_NAME-$BENCHMARK_NAME

# build all benchmark
make build-$FUZZER_NAME-all
```





## Instance

tbc: 增加一个实际 fuzzer 作为示例 ...

