---
layout: post
title: "Docker container常用命令"
subtitle: "介绍与container相关的常用命令"
date: 2022-04-25
author: "Cheney.Yin"
header-img: "img/bg-material.jpg"
tags: 
 - State Machine
 - Docker
 - container
 - nsenter
 - docker import
 - docker export
---

# Docker container常用命令

## 1 docker container状态图
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/resources/docker-container-state.svg" width = "75%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      图1 docker容器状态变化
  	</div>
</center>

## 2 命令详解

### 2.1 容器启动

首先创建容器，

```shell
docker create -it --name=busybox busybox
2dad99a1a04cd4adccfad768ac19389acbba4362090087fec8f587d5b9874abd
docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED         STATUS    PORTS     NAMES
2dad99a1a04c   busybox   "sh"      5 seconds ago   Created             busybox
```

docker容器处于`Created`状态，并未运行。

然后，执行`docker start`启动容器。

```shell
docker start busybox
busybox
docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED         STATUS         PORTS     NAMES
2dad99a1a04c   busybox   "sh"      2 minutes ago   Up 3 seconds             busybox
```

现在，容器处于运行状态。

这个过程可以通过`docker run`简化过程。

```shell
docker run -it --name busybox busybox
```

Docker后台启动流程为：

- Docker检查本地是否存在busybox镜像，如果镜像不存在则从Docker Hub拉取镜像。
- 使用busybox镜像创建并启动一个容器。
- 分配文件系统，并且在镜像只读层外创建一个读写层。
- 从Docker IP池中分配一个IP给容器。
- 执行用户启动命令运行镜像。

上述命令中，`-t`参数作用是分配一个伪终端，`-i`参数表示打开`STDIN`，因此`-it`表示进入交互模式。在交互模式下可以输入命令，

```shell
ps -a
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    8 root      0:00 ps -a
```

可以看到`PID`为1的进程`sh`。在交互模式下执行`exit`，意味着杀死了1号进程，等同于容器被杀死。**杀死容器主进程，则容器也会被杀死**。

```shell
/ # ps -a
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    8 root      0:00 ps -a
/ # exit
docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED         STATUS                     PORTS     NAMES
e27bbc62b96c   busybox   "sh"      3 minutes ago   Exited (0) 4 seconds ago             busybox
```

### 2.2 终止容器

使用`docker stop`命令终止容器。

```shell
docker stop --help

Usage:  docker stop [OPTIONS] CONTAINER [CONTAINER...]

Stop one or more running containers

Options:
  -t, --time int   Seconds to wait for stop before killing it (default 10)
```

改命令首先向运行中的容器发送`SIGTERM`信号，如果容器内1号进程接受并能处理，则等待1号进程处理完毕后退出；如果等待一段时间后，容器仍然没有退出，则会发送`SIGKILL`强制终止容器。

```shell
docker stop busybox
busybox

docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED          STATUS                       PORTS     NAMES
e27bbc62b96c   busybox   "sh"      15 minutes ago   Exited (137) 5 seconds ago             busybox
```

### 2.3 进入容器

进入运行状态的容器可以通过`docker attach`、`docker exec`、`nsenter`等多种方式进入。

（1）使用`docker attach`进入容器

```shell
# tm1
docker attach busybox
echo 'Hello'
Hello

# tm2
docker attach busybox
echo 'Hello'
Hello
```

`docker attach`命令同时在多个终端运行时，所有的终端将会同步显示相同内容，当某个命令行窗口阻塞时，其它命令行窗口也将无法操作。 

（2）使用`docker exec`进入容器

```shell
docker exec -it busybox /bin/sh
ps -a
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    7 root      0:00 /bin/sh
   14 root      0:00 ps -a
```

进入容器后，可以看到容器内有两个`sh`进程，7号进程是以`exec`的方式进入容器启动的，每个命令窗口都是独立互不干扰的。

（3）使用`nsenter`进入容器

[参考](https://mozillazg.com/2020/04/nsenter-usage.html)

`nsenter` 可以访问另一个进程的名称空间，并执行程序。

首先获取容器的主进程`PID`，

```shell
docker inspect -f \{\{.State.Pid\}\} busybox
11147
```

获取进程`PID`后，使用`nsenter`进入容器。

```shell
sudo nsenter --target 11147 --mount --pid --uts --ipc --net '/bin/sh'
/bin/ps -a
PID   USER     TIME  COMMAND
    1 root      0:00 sh
   22 root      0:00 /bin/sh
   23 root      0:00 /bin/ps -a
```

或者，

```shell
sudo nsenter --target 11147 --all '/bin/sh'
```

### 2.4 删除容器

删除停止状态的容器，

```shell
docker rm busybox
```

强制删除运行中的容器，

```shell
docker rm -f busybox
```

### 2.5 导出导入容器

（1）导出

`docker export`命令导出一个容器到文件，不管容器是否处于运行状态。

```shell
docker export busybox > busybox.tar
```

容器被打包进了`busybox.tar`文件。

（2）导入

`docker import`命令可将`docker export`导出文件转化为本地镜像，最后使用`docker run`重新创建启动新的容器。

```shell
docker import busybox.tar busybox:test
```



