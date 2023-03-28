---
layout: post
title: "Some tricks about shell script"
subtitle: "This is a subtitle"
date: 2023-03-28
author: "Cheney.Yin"
header-img: "img/bg-material.jpg"
tags:
 - Shell
 - Linux
 - set
 - shopt
 - pipefail
---

# 一些Shell脚本编写技巧

## `set -o errexit`
命令`set -o errexit`可以简化为`set -e`。其作用是在脚本出现错误时，退出执行。（*默认并不退出*）

### 未使用配置
```shell
#!/bin/bash
# foo为不存在的命令
foo
echo 'bar'
```
输出，
```shell
./errexit.sh: line 4: foo: command not found
bar
```

### 使用配置
```shell
#!/bin/bash
set -o errexit
# foo为不存在的命令
foo
echo 'bar'
```
输出，
```shell
./errexit_after.sh: line 4: foo: command not found

```
这证明`echo 'bar'`未执行。

## `set -o nounset`
命令`set -o nounset`可简化为`set -u`。其作用是在处理未定义变量时，直接报错。（*默认，是将未定义变量视为空字符串。*）

### 未使用配置

```shell
#!/bin/bash

v1='OK'
echo "v0=$v0"
echo "v1=$v1"
#!/bin/bash
```
输出，
```shell
v0=
v1=OK
```

### 使用配置

```shell
#!/bin/bash
set -o nounset

v1='OK'
echo "v0=$v0"
echo "v1=$v1"
#!/bin/bash
```
输出，
```shell
./nounset_after.sh: line 5: v0: unbound variable
```

## `set -o pipefail`
其作用是，在整个管道操作中，任意一个命令失败，整个管道操作将失败。（*默认情况下，Bash只检查最右侧的命令返回值，最右侧命令成功则认为整个管道操作成功。*）

### 未使用配置
```shell
#!/bin/bash

foo | echo 'OK'
echo $?
```
输出，
```shell
OK
./pipefail_before.sh: line 3: foo: command not found
0
```

### 使用配置
```shell
#!/bin/bash
set -o pipefail

foo | echo 'OK'
echo $?
```
输出，
```shell
OK
./pipefail_after.sh: line 4: foo: command not found
127
```
返回的是127而非0，说明管道失败。

一般该配置项会与`set -o errexit`一起使用，失败后直接退出。
```shell
#!/bin/bash
set -o pipefail -o errexit

foo | echo 'OK'
echo $?
```
输出，
```shell
OK
./pipefail_after.sh: line 4: foo: command not found
```
没有输出返回值，而是直接退出。

## `set -x`
其作用是打印执行命令。
```shell
#!/bin/bash
set -x

v1='OK'
echo "v0=$v0"
echo "v1=$v1"
```
输出，
```shell
+ v1=OK
+ echo v0=
v0=
+ echo v1=OK
v1=OK
```

## `shopt -s nullglob`

目录下存在`errexit_after.sh`、`errexit_before.sh`两个文件，可以通过如下命令匹配打印。
```shell
echo err*.sh

#输出
errexit_after.sh errexit_before.sh
```
目录下不存在`lerr*.sh`的文件，
```shell
echo lerr*.sh

#输出
lerr*.sh
```
由于无法匹配，将匹配串当做字符串处理。
开启`shopt -s nullglob`配置后，对于无法匹配的结果，作为空来处理。
```shell
shopt -s nullglob
echo lerr*.sh

#输出

```
返回空串。

