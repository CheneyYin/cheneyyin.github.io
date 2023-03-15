---
layout: post
title: "Docker image常用命令"
subtitle: "介绍与docker image相关的常用命令"
date: 2022-05-12
author: "Cheney.Yin"
header-img: "img/bg-material.jpg"
tags:
 - Docker
 - docker image
 - docker image inspect
 - overlay2
---

# Docker image常用命令

# 1 简介

`docker image`命令用于管理镜像，常见使用方式如下。

```shell
❯ docker image --help

Usage:  docker image COMMAND

Manage images

Commands:
  build       Build an image from a Dockerfile
  history     Show the history of an image
  import      Import the contents from a tarball to create a filesystem image
  inspect     Display detailed information on one or more images
  load        Load an image from a tar archive or STDIN
  ls          List images
  prune       Remove unused images
  pull        Pull an image or a repository from a registry
  push        Push an image or a repository to a registry
  rm          Remove one or more images
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
```

## 2 image inspect

示例使用的**say**镜像的**Dockerfile**如下：

```dockerfile
FROM scratch
WORKDIR /
COPY ./say /
CMD ["/say"]
```

这里，着重梳理一下`inspect`。`inspect`用于查看镜像的详细信息。

```shell
❯ docker image inspect say
[
    {
        "Id": "sha256:aa5bb1e797b5648a5c61618c0b6171a25189a2d6e18ec37b9466743a35945ab9",
        "RepoTags": [
            "say:latest"
        ],
        "RepoDigests": [],
        "Parent": "sha256:378b8f2564caa9bf6903179a94ae392776f77c2652dec3327889b34bfc742272",
        "Comment": "",
        "Created": "2021-11-05T05:02:17.489994051Z",
        "Container": "5acca3b69c9336715e078374b8cc00625325d1e93c0045be064d446ca1514e7e",
        "ContainerConfig": {
            ... ...
            "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) ",
                "CMD [\"/say\"]"
            ],
            "Image": "sha256:378b8f2564caa9bf6903179a94ae392776f77c2652dec3327889b34bfc742272",
            "Volumes": null,
            "WorkingDir": "/",
            ... ...
        },
        "DockerVersion": "20.10.9",
        "Author": "",
        "Config": {
            ... ...
            "Cmd": [
                "/say"
            ],
            "Image": "sha256:378b8f2564caa9bf6903179a94ae392776f77c2652dec3327889b34bfc742272",
            ... ...
        },
        ... ...
        "GraphDriver": {
            "Data": {
                "MergedDir": "/var/lib/docker/overlay2/de4b43fcc2e259b223c2e5a29c5883711aa15ca0b127e8df2482bfc00ed4e84c/merged",
                "UpperDir": "/var/lib/docker/overlay2/de4b43fcc2e259b223c2e5a29c5883711aa15ca0b127e8df2482bfc00ed4e84c/diff",
                "WorkDir": "/var/lib/docker/overlay2/de4b43fcc2e259b223c2e5a29c5883711aa15ca0b127e8df2482bfc00ed4e84c/work"
            },
            "Name": "overlay2"
        },
       ... ...
    }
]
```

这里，关注`Parent`、`GraphDriver`。

`Parent`指的是该镜像的基础镜像，docker的镜像是一层一层叠加而来的，没执行**Dockerfile**中的一条指令就会产生一个层镜像，可以使用`docker image history`查看镜像的叠加关系。

```shell
❯ docker image history say
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
aa5bb1e797b5   2 hours ago   /bin/sh -c #(nop)  CMD ["/say"]                 0B
378b8f2564ca   2 hours ago   /bin/sh -c #(nop) COPY file:0ca1aaab19fa1703…   923kB
81ec91d2bac1   2 hours ago   /bin/sh -c #(nop) WORKDIR /                     0B
```

`inspect`信息中的`Parent: sha256:378b8f2564ca ... ...`，同第二层镜像（`378b8f2564ca   2 hours ago   /bin/sh -c #(nop) COPY file:0ca1aaab19fa1703… `即**COPY**操作）对应。

`GraphDriver.Data`指向目录`/var/lib/docker/overlay2/de4b43fcc2e259b223c2e5a29... ...`，该目录是镜像的**overlay2**存储目录。

```shell
/var/lib/docker/overlay2/de4b43fcc2e259b223c2e5a29c5883711aa15ca0b127e8df2482bfc00ed4e84c
├── committed
├── diff
│   └── say
└── link
```

第二层镜像`378b8f2564ca`的**overlay2**存储目录同样指向目录`/var/lib/docker/overlay2/de4b43fcc2e259b223c2e5a29... ...`，这是因为第三层镜像是执行`CMD`得到的，没有更改镜像上的任何文件。

```shell
docker image inspect 378b8 | grep 'overlay2'
                "MergedDir": "/var/lib/docker/overlay2/de4b43fcc2e259b223c2e5a29c5883711aa15ca0b127e8df2482bfc00ed4e84c/merged",
                "UpperDir": "/var/lib/docker/overlay2/de4b43fcc2e259b223c2e5a29c5883711aa15ca0b127e8df2482bfc00ed4e84c/diff",
                "WorkDir": "/var/lib/docker/overlay2/de4b43fcc2e259b223c2e5a29c5883711aa15ca0b127e8df2482bfc00ed4e84c/work"
            "Name": "overlay2"
```

而第一层镜像`81ec91d2bac1`的`Parent`和`GraphDriver.Data`均为**空**，这是应为**say**镜像是`FROM scratch`，`scratch`是空镜像。

```shell
docker image inspect 81ec91d2bac1
... ...
"Parent": "",
... ...
"GraphDriver": {
            "Data": null,
            "Name": "overlay2"
        },
... ...
```

