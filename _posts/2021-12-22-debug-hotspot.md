---
layout: post
title: "调试Hotspot"
subtitle: "记录调试Hotspot的环境和过程"
date: 2021-12-22
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags: 
 - Java
 - JVM
 - Hotspot
 - Docker
 - OpenJDK
 - debug
 - GDB
 - debuginfo
---

# 调试Hotspot

## 容器环境

```shell
docker run -itd -v /home/cheney/expr/jvm/openjdk12/:/opt/ --hostname openjdk.com --name openjdk ubuntu:18.04 /bin/bash
```

安装依赖

```shell
apt install build-essential
apt install zip unzip
apt install openjdk-11-jdk
apt install libfreetype6-dev libcups2-dev libasound2-dev libffi-dev
apt install libx11-dev libxext-dev libxrender-dev libxrandr-dev libxtst-dev libxt-dev
apt install autoconf
apt install file libfontconfig1-dev
```

## 编译

配置`debug`编译模式，

```shell
bash configure \
--with-debug-level=slowdebug \
--with-native-debug-symbols=internal \
--with-jvm-variants=server \
--with-num-cores=4 \
--with-jobs=4 \
--with-log=info


```

成功，则输出，

```shell
====================================================
A new configuration has been successfully created in
/opt/openjdk/build/linux-x86_64-server-fastdebug
using configure arguments '--with-debug-level=fastdebug'.

Configuration summary:
* Debug level:    fastdebug
* HS debug level: fastdebug
* JVM variants:   server
* JVM features:   server: 'aot cds cmsgc compiler1 compiler2 epsilongc g1gc graal jfr jni-check jvmci jvmti management nmt parallelgc serialgc services shenandoahgc vm-structs zgc'
* OpenJDK target: OS: linux, CPU architecture: x86, address length: 64
* Version string: 12-internal+0-adhoc..openjdk (12-internal)

Tools summary:
* Boot JDK:       openjdk version "11.0.11" 2021-04-20 OpenJDK Runtime Environment (build 11.0.11+9-Ubuntu-0ubuntu2.18.04) OpenJDK 64-Bit Server VM (build 11.0.11+9-Ubuntu-0ubuntu2.18.04, mixed mode, sharing)  (at /usr/lib/jvm/java-11-openjdk-amd64)
* Toolchain:      gcc (GNU Compiler Collection)
* C Compiler:     Version 7.5.0 (at /usr/bin/gcc)
* C++ Compiler:   Version 7.5.0 (at /usr/bin/g++)

Build performance summary:
* Cores to use:   8
* Memory limit:   11862 MB
```

执行编译

```shell
make images
```

## 解决GDB SIGSEGV问题

创建`～/.gdbinit`，

```shell
cat ~/.gdbinit
handle SIGSEGV pass noprint nostop
handle SIGBUS pass noprint nostop
```

## 查看debuginfo文件

```shell
readelf -wi ./javac.debuginfo | less
```

待续

