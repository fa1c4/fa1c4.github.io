---
title: 'Shell Programming'
date: 2023-10-20
permalink: /posts/2023/10/shell-programming/
tags:
  - shell
  - script
---

# Shell Programming

## Init

Interpreter by default starts with `#!`

```bash
#!/bin/bash 
```

Execution of  shell script

```bash
# method1: use absolute or relavant path (needs +x authority)
./script.sh

# method2: use /bin/sh or /bin/bash 
# (no need for +x authority, because bash creates new shell to execute script)
bash ./script.sh

# method3: source or . 
# this method will create sub-shell, env variant inherits from upper-shell
source script.sh
. script.sh
```



## Variant

Predefined Variants of System

```bash
$HOME, $PWD, $SHELL, $USER, # etc
```

print all system env varients

```bash
> env
> echo $env-var
> printenv
```



self-define variants

```shell
variant=value # note: there is no blankspace besides '='
variant='string exmple'
variant="string exmple"
variant=1+2 # echo variant -> 1+2
variant=$((1+2)) # echo variant -> 3
variant=$[1+2] # echo variant -> 3
readonly variant=value # set variant readonly, which can't be modified or be unset
```



special variants

```shell
$n # stands for n-th args parameter
echo $0 $1 $2
./script.sh 1stparameter 2ndparameter
# script-name 1stparameter 2ndparameter

# special usage
$# # stands for count of parameters
$* # stands for all parameters as one obj
$@ # stands for all parameters as each obj
$? # stands for return value of last execution 
```



here is an example to show the difference between `$*` and `$@`

```bash
# shell code 
echo "---- \$* ----"
for n in "$*" # note that must use "" to see the difference between $* and $@
do
    echo $n
done

echo "---- \$@ ----"
for n in "$@" # note that not use "" there is no difference 
do
    echo $n
done   

# bash command
./script.sh a b c d e
---- $* ----
a b c d e
---- $@ ----
a
b
c
d
e
```





## Operator

arithmetic operators

```shell
expr 1 + 2
expr 1 - 2
expr 1 \* 2
expr 1 / 2
```



popular usage

```shell
$((expression)) or $[expression]
```



expr assignment

```shell
variant=$(expr 1 + 2)
variant=`expr 1 \* 2`
variant=$[1 * 2] # no needs add escape character before * in []
```



new support expr

```shell
let n += 1
let n++
# use let before expr can apply normal program language syntax
```





## Test Condition

```shell
# normal usage test
var="stringcmp"
test $var = "stringcmp"
echo $?

# easy usage test 
var="stringcmp"
[ $var = "stringcmp" ] # note that there are 2 blankspace in [  ] 
```

Besides, there is no `<` or `>` test condition in shell 

>TEST CONDITIONS using in shell
>
>-eq equal
>
>-ne not equal
>
>-gt greater than
>
>-lt less than
>
>-le less equal
>
>-ge greater equal



```shell
[ $v1 -lt $v2 ]
[ $v1 -gt $v2 ]
[ $v1 -le $v2 ]
[ $v1 -ge $v2 ]
```



check authority of shell file

```shell
[ -r script.sh ]
[ -w script.sh ]
[ -x script.sh ]
```



check class of files

```shell
[ -e file ] # obj exists or not
[ -f file ] # normal file exists or not
[ -d directory ] # is a directory or not
```



multiple conditions

`&&` stands for `and`

`||` stands for `or`

```shell
[ condition1 ] && [ condition2 ] || [ condition3 ]
```

`-a` stands for `and`

`-o` stands for `or`

```shell
[ condition1 -a condition2 -o condition3 ]
```





## Process Control

`if`  statement

```shell
# single branch
if [ condition ];then # ';' spilits the execution of commands
	# code here
fi

# or
if [ condition ]
then
	# code here
fi


# multiple branch
if [ condition1 ]
then 
	# code here
elif [ condition2 ]
then	
	# code here
elif [ condition3 ]
then
	# code here
...
else # note that there is no 'then' after else 
	# code here
fi
```



`case` statement

```shell
case $variant in 
v1) # case v1
	# code here
;; # double ';' to end one branch
v2) # case v2
	# code here
;;
v3) # case v3
	# code here
;;
...
*) # stands for default
	# code here
;;
esac
```



`for` statement

```shell
for ((initial;loop-condition;variant-mutation))
do
	# code here
done

# example code
sum=0
for (( i=0; i<10; i++ )) # note that in (()) can use < > <= >= 
do
    sum=$[$sum+$i] # note that $[] to calculate
                   # if not use $[] will be string concat
done
echo sum=$sum


# other usage
for n in 1 2 3 4 5
do
	echo $n
done

sum=0
for n in {1..100} # note that there are only 2 '.'
do
	sum=$[$sum+$n]
done
echo $sum
```



`while` statement

```shell
while [ condition ]
do
	# code here
done

# or 
while ((condition))
do
	# code here
done
```







## Input

`read` statement

> -p: sets the prompt for reading the value
>
> -t: Indicates the amount of time (in seconds) to wait for the value to be read. The default is always



```shell
read -t 10 -p "input your string (in 10 seconds)" string
```





## Functions

System function call

```shell
# example: basename
basename [string / pathname] [suffix] 
# basename will remove all prefix-str including '/' in string or path name
# if suffix is indicated then suffix-str in string or pathname will be removed
```



self-defined function 

```shell
[function] funcname[()] # parameters are $1, $2, ..., $n
{
	# code here
	[return int;]
}
```

We can think of functions as scripts. Some tips here:

+ Before call function in shell script, we have to define it first
+ Return value of function can only be referenced as `$?`; if add `return intval` statement in the last line of function, the return value of function is `intval` whose range is between 0 to 255.



example of self-define function

```shell
function sum_of_nums()
{
    sum=0
    for n in $@
    do
        sum=$[$sum+$n]
    done
    echo $sum
    return 0;
}

if [ $# -lt 2 ]
then
    sum_of_nums 1 2 3 4 5 # echo 15
else 
    sum_of_nums $1 $2 # echo $[$1 + $2]
fi

res=$(sum_of_nums $1 $2) # return value to main
```





## Practice

Descriptions: Set a `dirname` (there is no '/' in suffix), tar all files in the `dirname` everyday, and attach the date after `.tar` filename then save to `/root/archive`

> tar
>
> -c: stands for archive
>
> -z: stands for compress, the compressed file suffix is .tar.gz

shell script implementation

```shell
# tar synthesis example 
if [ $# -ne 1]; then
    echo "Usage: $0 <dir>"
    exit 1
fi

# get the path of the dir from the first argument
if [ -d $1 ]; then
    dir=$(cd $1; pwd)
else
    echo "$1 is not a dir"
    exit 1
fi

DIR_NAME=$(basename $dir)
DIR_PATH=$dir

# get the current time 
DATE=$(date +%Y%m%d)

# set the name of the tar file
FILE=archive_$DIR_NAME_$DATE.tar.gz
# DEST=/root/archive/$FILE # we test in the /tmp dir
DEST=/tmp/archive/$FILE

# create the dir of the tar file if not exists
if [ ! -d $(dirname $DEST) ]; then
    mkdir -p $(dirname $DEST)
    echo "create dir $(dirname $DEST) successfully"
fi

# start to tar
echo "start to tar $DIR_PATH to $DEST"
tar -zcvf $DEST $DIR_PATH

if [ $? -eq 0 ]; then
    echo "tar success"
else
    echo "tar failed"
    exit 1
fi

exit

# set daily backup
# crontab -e
# 0 0 * * * /root/script.sh /root/dir
# the first 0 is minute, the second 0 is hour, 
# the third * is day, the fourth * is month, the fifth * is week
```





## RegEx

normal matching

```shell
grep dest-string
```



special characters

> ^ : the start of matching
>
> $ : the end of matching
>
> . : any single character
>
> \* : combines with pre-one char to math any character any times
>
> [n1-n2,c1-c2] : match range n1 to n2 or c1 to c2
>
> \ : escape character, used like \\$, to match \$

note that: `^$` will match null. There is no blank space in [n1-n2,c1-c2]

`grep -n ^$` will match all null lines

`grep ^start.*end$` will match those string that starts with `start` and ends with `end`

`grep -E ^1[34578][0-9]{9}$` to match normal phone number, `-E` stands for supports extensive usage, grep doesn't support `{}` usage by default





## String Process

`cut` usage

```shell
cut [parameters] filename
# separator is tab by default

# example: extracts 2,3 cols from source.txt and the separator for columns is " "
cut -d " " -f 2,3 source.txt

# example: extracts 1,2,3 cols from /etc/passwd for those lines end with bash, and the separator is ":"
cat /etc/passwd | grep bash$ | cut -d ":" -f 1,2,3

# example: extracts 6 col and all cols after it from /etc/passwd for those lines end with bash, and the separator is ":"
cat /etc/passwd | grep bash$ | cut -d ":" -f 6-

# example: cut the IP address
ifconfig ens33 | grep netmask | cut -d " " -f 10
```

> -f : extracts column
>
> -d : indicates separator
>
> -c : splits as char, use -c n to fetch n-th column





`awk` usage

```shell
awk [parameters] '/pattern1/{action1} /pattern2/{action2}...' filename
# pattern stands for matching pattern
# action stands for execution commands during matching

# first explain the data structure of /etc/passwd 
# user-name:password(encrypted):user-id:group-id:comment:user-homedir:shell-interpreter
# match lines in /etc/passwd for those starts with 'root' then output the 7th column of the lines
cat /etc/passwd | awk -F ":" '/^root/{print $7}'
# comparing to grep usage
cat /etc/passwd | grep ^root | cut -d ":" -f 7

# print 1,7 cols
cat /etc/passwd | awk -F ":" '/^root/{print $1", "$7}'

# use variants
awk -v i=2 -F ":" '{print $3+i}'

# inner variants
# FILENAME, NR, NF
awk -F ":" '{print "filename: "FILENAME " row number: "NR " col number: " NF}' /etc/passwd

# print row number of all null lines 
ifconfig | awk -F " " '/^$/{print NR}'
```

> -F : indicates separator of input filename
>
> -v : assignment for a self-defined variant





## Summary

So this is, the blog is about basic shell programming, including 

+ Variants
+ Expr
+ Conditions
+ Process Control
+ RegEx
+ String Process