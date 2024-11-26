---
title: "Shell Programming Techs I"
date: 2024-11-25
tags:
  - shell
  - script
---

# Shell Programming Techs I

记录阅读 [MAGMA](https://github.com/HexHive/magma) 源码时，遇到的一些高级shell编程技巧

## Set

设置环境变量

```shell
set -a # 激活 Shell 的 allexport 模式。当 allexport 模式启用时，所有定义的变量（包括后续加载的变量）都会自动导出为环境变量。
source "$1" # 加载配置文件，将其中定义的变量和命令导入当前 Shell 环境。
set +a # 关闭 allexport 模式。
```



设置shell变量

```shell
MAGMA=${MAGMA:-"$(cd "$(dirname "${BASH_SOURCE[0]}")/../../" >/dev/null 2>&1 && pwd)"}
```

`${MAGMA:-value}`: 表示如果环境变量 `MAGMA` 已经被定义，则保留其原值。如果 `MAGMA` 未被定义，则设置为后面指定的值。

`$(...)`: 用于命令替换，表示运行括号中的命令并将其输出结果赋值给变量。

`dirname "${BASH_SOURCE[0]}"`: 返回当前脚本文件所在的目录。`${BASH_SOURCE[0]}` 是当前脚本文件的路径。

`>/dev/null 2>&1`: 将标准输出重定向到 `/dev/null`，抑制命令的标准输出和错误输出。`2>`表示重定向标准错误。`&1`表示将标准错误的输出流合并到标准输出流中。避免命令的输出污染终端显示。仅判断命令是否执行成功（通过 `$?` 查看退出状态），不关心输出内容。

相应的有，`&> /dev/null` 抑制输出，避免干扰。



## Operation

获取CPU列表

```shell
WORKERS_ALL=($(lscpu -b -p | sed '/^#/d' | sort -u -t, -k ${WORKER_MODE}g | cut -d, -f1))
```

`lscpu -b -p`: 列出 CPU 的简洁信息，以逗号分隔。

`sed '/^#/d'`: 删除以 `#` 开头的注释行。

`sort -u -t, -k ${WORKER_MODE}g`: 按第 `WORKER_MODE` 列对数据进行排序并去重。

`cut -d, -f1`: 提取第一个字段（CPU 核心编号）。

`(...)`: 将命令的输出分割为多个元素（基于空白符）。



确保绝对路径

```shell
WORKDIR="$(realpath "$WORKDIR")"
```

双引号用于避免路径中可能包含的空格或特殊字符导致问题。

`$()` 用于捕获子命令的输出。



安全删除文件

```shell
shopt -s nullglob # shopt 是 Bash 的一个内建命令，用于启用或禁用 shell 选项。nullglob 是一个特定的 shell 选项，其作用是控制文件名通配符（如 *）在没有匹配文件时的行为。启用时，文件名通配符在没有匹配文件时返回空字符串。启用 nullglob 确保目录 $LOCKDIR 中没有匹配文件时，通配符 * 不会被当作字符串 *，而是返回空。
rm -f "$LOCKDIR"/* # 如果 $LOCKDIR 是空目录，启用了 nullglob 的情况下，* 返回空，rm 不会执行删除操作，也不会报错。
shopt -u nullglob # 关闭 nullglob 后恢复 Bash 的默认行为，即文件名通配符在没有匹配文件时不返回空，而是原样保留。
```



排序操作

```shell
cids=($(sort -n < <(basename -a "${campaigns[@]}")))
```

`sort -n`: 按数字排序这些基名。

`< <(...)`: 将子命令的输出重定向为标准输入流。



导出函数

```shell
export -f get_next_cid # 将函数导出为当前 shell 的子进程可用（例如在 bash 脚本或其他子 shell 中调用）。
```



清理临时文件

```shell
trap 'rm -f "$LOCKDIR/$mux"' EXIT
```

`trap`: 设置退出时的清理动作。捕获 `EXIT` 信号，当脚本或函数退出时（无论正常还是异常退出），都会执行指定命令。



移除参数

```shell
shift # 使用 shift 移除第一个参数，将后续参数（命令及其参数）从 $2..N 调整为 $1..N。
```



创建子shell

```shell
(
  flock -xF 200 &> /dev/null
  "${@}"
) 200>"$LOCKDIR/$mux"
```

使用 `()` 启动一个子 shell，确保锁的范围仅限于当前子 shell。

`flock` 是一个用于管理文件锁的工具。`-x`: 获取排他锁（独占锁），确保同一时间只有一个进程持有该锁。`-F`: 强制刷新文件，如果文件描述符无法获取锁时立即返回（不会阻塞）。

`200>"$LOCKDIR/$mux"` 将文件描述符 `200` 定向到 `$LOCKDIR/$mux`，该文件用于作为锁标识。

`"${@}"` 运行给定命令，`@` 是传递给 `mutex` 函数的命令及其参数。



失败处理

```shell
|| if [ $? -eq $errno_lock ]; then
    continue
fi
```

`$?` 表示前一个命令的退出状态码。



文件操作：防止覆盖已有文件

```shell
set -o noclobber # 确保文件创建是原子的（即如果文件已存在则创建失败）。
```



`cut`: 一个用于从文本中按字段或字符位置提取数据的工具

```shell
cut -d',' -f2- <<< $WORKERSET

# example
cut -d',' -f2- <<< "a,b,c"
echo "a,b,c" | cut -d',' -f2- # equivalent command
# output: a b c
```

`-d','`: 指定逗号 `,` 为字段分隔符。

`-f2-`: 提取从第 2 个字段开始的所有字段。



后台任务处理

```shell
for job in `jobs -p`; do
    if ! wait $job; then
        continue
    fi
done
```

`jobs -p` 列出当前 Shell 中的所有后台任务的进程 ID（PID）。

`wait $job` 等待后台任务完成。如果任务成功完成，`wait` 返回 0；否则返回非零状态码。



遍历目录下所有文件

```shell
find "$LOCKDIR" -type f | while read lock; do
    if inotifywait -qq -e delete_self "$lock" &> /dev/null; then
        continue
    fi
done
```

`find "$LOCKDIR" -type f` 找出 `$LOCKDIR` 所有文件

`while read lock` 逐个遍历文件设为 `lock`



变量获取

```shell
set -- "DEFAULT" "${@:2}"
set -- "${@:2}" # 删除第一个参数（包括 DEFAULT），使用剩余参数。
```

`"DEFAULT"` 替换第一个参数，生成新的参数列表。`"${@:2}"` 代表原参数的第 2 个及之后的部分。





## Array

数组切片

```shell
WORKER_POOL="${WORKERS_ALL[@]:0:WORKERS}"
```

`[@]` 表示操作数组的所有元素。如果只用 `${#WORKERS_ALL}`，则表示字符串长度，而不是数组元素数量。`${#array[@]}` 语法**只能**用于明确**声明为数组**的变量。

`[@]:start:length` 是 Bash 中的数组切片语法，用于提取数组的一部分。`start` 表示起始索引（从 `0` 开始），`length` 表示要提取的元素个数。

`"${...}"` 将数组切片的结果（即前 `WORKERS` 个元素）合并成一个字符串，元素之间用空格分隔。



使用 `read` 将字符串解析为数组

```shell
IFS=',' # 设置字段分隔符
read -a workers <<< "$AFFINITY"
```

`read -a`: 读取输入并将其存储为一个数组。

`<<<` 是 Here String 的语法，用于将一个字符串作为命令的标准输入。



数组定义与解引用

```shell
name="${name}[@]" # 将变量名作为数组处理
value="${!name}" # 间接引用变量 name 的值
```





## Functions

将传入的参数用指定分隔符连接成一个字符串

```shell
function join_by { local IFS="$1"; shift; echo "$*"; } # 定义分隔符为"$1"，移除第一个参数，打印每个参数

pattern=$(join_by _ "${@}") # ${@} 包含传入的所有参数
```









