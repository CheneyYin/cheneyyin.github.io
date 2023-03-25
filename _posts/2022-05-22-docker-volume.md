---
layout: post
title: "Docker数据卷"
subtitle: "介绍Docker数据卷的相关概念和使用方法"
date: 2022-05-22
author: "Cheney.Yin"
header-img: "//imgloc.com/i/iADI63"
tags:
 - Docker
 - 数据卷
---

# docker数据卷

## 管理数据卷

```she
docker volume help

Usage:  docker volume COMMAND

Manage volumes

Commands:
  create      Create a volume
  inspect     Display detailed information on one or more volumes
  ls          List volumes
  prune       Remove all unused local volumes
  rm          Remove one or more volumes
```

创建数据卷，

```shell
docker volume create one-volume
one-volume
```

查看数据卷，

```shell
docker volume ls
DRIVER    VOLUME NAME
local     4d8f935f757dbb8ac4b10430f5aab5ca75da0858dd1baa71918ea8c7ad07cc2a
local     9c68e6e515d217439ba91b7afd022015404f26ca3986463823b3e470394157cd
local     9f59429de397d54c782fa11ae6cd2107079f97ba91b7008e2b5a541b0b1f4b95
local     78e96e66f3fdef7fd4e316400a08f518ac6568f84704ca388d77a82b0a8f1509
local     01714f7baa8968d17ea4b17b2fea9622463bfc43a42c15f4625e67ec0ca69181
local     bc2041319f5b1d890902fd3d677efcd38730ee60d92f805e25318c01a1a8b740
local     f5a6a2826e5f5f455b47fb563856a6947a8b74c77f5a7b4c10e8547c3fceccf2
local     one-volume
```

## 容器配置数据卷

启动容器`ubuntu`，并指定数据卷，

```shell
docker run -itd --mount source=one-volume,target=/opt --hostname ubuntu.com --name ubuntu ubuntu /bin/bash
8bde099b742cc9702c9cb1b939e2de933f793afadafb587a2b0e1f0c253959c7
```

```shell
docker inspect --format '\{\{.Mounts\}\}' ubuntu
[{volume one-volume /var/lib/docker/volumes/one-volume/_data /opt local z true }]
```

向`/opt/hello`写入`hello`，

```shell
docker exec -it ubuntu /bin/bash -c 'echo "hello" > /opt/hello'
docker exec -it ubuntu /bin/bash -c 'cat /opt/hello'
hello
```

## 容器复用容器数据卷

启动容器`busybox`，并从容器`busybox`复用数据卷，

```shell
docker run -itd --volumes-from ubuntu --hostname busybox.com --name busybox busybox /bin/sh
638d959856e08b73e14bb9d757d5512f6a6da5f132b44c9d0c03020a27e8fcd4
```

```shell
docker inspect --format '\{\{.Mounts\}\}' busybox
[{volume one-volume /var/lib/docker/volumes/one-volume/_data /opt local  true }]
```

访问共享文件，

```shell
docker exec -it busybox /bin/sh -c 'cat /opt/hello'
hello
```

## 数据卷映射

在`docker run`中，使用`-v`开启容器数据卷，

```shell
docker run -itd -v /opt --hostname busybox.com --name busybox busybox /bin/sh
9a4900b2a5f69037f33a57a02858a9f3b9c25bd4ae799e3ea2008ce17b1fc688
```

```shell
docker inspect --format '\{\{.Mounts\}\}' busybox
[{volume 941a2e4bdf9f76c4e07c0b3fd67688495b384f8b9ecfa02332bb8ec1e369ac31 /var/lib/docker/volumes/941a2e4bdf9f76c4e07c0b3fd67688495b384f8b9ecfa02332bb8ec1e369ac31/_data /opt local  true }]
```

`-v HOST_PATH:CONTAINER_PATH`可以指定宿主机路径作为数据卷，

```shell
docker run -itd -v /home/XXX/expr/docker/A:/opt --hostname busybox.com --name busybox busybox /bin/sh
5584070ff88e4d62a3bb84fbf5978d9efe5a1185cbbca54b12bbad51ee3b9e78
```

```shell
docker inspect --format '\{\{.Mounts\}\}' busybox
[{bind  /home/cheney/expr/docker/A /opt   true rprivate}]
```

