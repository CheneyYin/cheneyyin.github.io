---
layout: post
title: "Make使用总结"
subtitle: "简单介绍了Make的使用基本方法"
date: 2022-04-09
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags:
 - make
 - makefile
 - build
---

# makefile介绍
## 1 书写规则
规则包括两部分：(1)依赖关系，(2)生成目标的方法。
### 语法
语法1
```shell
targets : prerequisites
    command
    ...
```
语法2
```shell
targets : prerequisites ; command
    command
    ...
```
- `targets`是文件名，可以是一个或者多个文件名（多个文件名以空格分割），可以是通配符。
- `command`如果不同`target : prerequisites`在同一行，则`command`必须以`TAB`开头。
- `prerequisites`是`targets`依赖的目标文件，如果依赖的目标文件比`targets`新，则`targets`会重新生成。
### 伪目标
- 伪目标是标签而不是文件。`make`无法生成它的依赖关系、决定它是否执行。
- 伪目标一般不能和文件同名，除非使用`.PHONY`指明为伪目标。
```shell
all : prog1 prog2 prog3
.PHONY : all

prog1 : prog1.o utils.o
    cc -o prog1 prog1.o utils.o

prog2 : prog2.o
    cc -o prog2 prog2.o

prog3 : prog3.o sort.o utils.o
    cc -o prog3 prog3.o sort.o utils.o

```
- 伪目标是标签而不是文件，所以为目标总是被执行。
```shell
.PHONY : cleanall cleanobj cleandiff

cleanall : cleanobj cleandiff
    rm prog

cleanobj ：
    rm *.o

cleandiff :
    rm *.diff
```
当执行`make cleanall`时，`cleanobj`和`cleandiff`总会被执行。
### 多目标
多个目标有相同依赖。
```shell
objects := $(wildcard *.c)
a b : $(objects)
    echo '$@ - $(objects)'
```
- `$@`为目标集合。

执行`make a b`输出如下：
```shell
make a
echo 'a A.c B.c'
a A.c B.c
make a
echo 'a A.c B.c'
b A.c B.c
```
上述示例等价于：
```shell
objects := $(wildcard *.c)
a : $(objects)
    echo 'a - $(objects)'
b : $(objects)
echo 'b - $(objects)'
```

待续