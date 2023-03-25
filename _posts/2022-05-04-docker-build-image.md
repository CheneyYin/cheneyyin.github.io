---
layout: post
title: "Docker构建镜像"
subtitle: "介绍如何构建一个简单的docker镜像"
date: 2022-05-04
author: "Cheney.Yin"
header-img: "//imgloc.com/i/iADI63"
tags:
 - Docker
 - docker image
 - docker build
 - Dockerfile
---

# Docker构建镜像

## 1 build镜像方式

`docker`提供的镜像构建方式包括两种：

- 使用`docker commit`命令从运行中的容器提交为镜像。
- 使用`docker build`命令从**Dockerfile**文件构建镜像。

第一种方式这里不做赘述，第二种方式是最重要、最常用的方式。

## 2 Dockerfile构建镜像

**Dockerfile**常用指令如下，

| **Dockerfile**指令 | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| `FROM`             | **Dockerfile**除了注释第一行必须是`FROM`，`FROM`后面跟镜像名称，代表我们要基于哪个基础镜像构建。 |
| `RUN`              | `RUN`后面跟一个具体的命令，类似于**Linux**命令行执行命令。   |
| `ADD`              | 拷贝本机文件或远程文件至镜像，压缩文件会被展开。             |
| `COPY`             | 拷贝本机文件至镜像内。                                       |
| `USER`             | 指定容器启动的用户。                                         |
| `ENTRYPOINT`       | 容器的启动命令。                                             |
| `CMD`              | `CMD`为`ENTRYPOINT`指令提供默认参数，也可以单独使用`CMD`指定容器启动参数。 |
| `ENV`              | 指定容器运行时的环境变量，格式为`key=value`。                |
| `ARG`              | 定义外部变量，构建镜像时可以使用`build-arg=`的格式传递参数用于构建。 |
| `EXPOSE`           | 指定容器监听的端口，格式`[port]/tcp`或者`[port]/udp`。       |
| `WORKDIR`          | 为**Dockerfile**中跟在其后的所有命令设置工作目录。           |

## 3 Tiny示例镜像制作

这里，为大家演示如何使用**Dockerfile**构建一个极为精简的镜像。

首先，编写一段名为`say.c`的代码，内容如下，

```c
#include <stdio.h>

int main() {
	char buf[128];
	scanf("%s", buf);
	printf("say:%s\n", buf);
	return 0;
}
```

执行编译。（这里使用了`--static`开启了静态链接，如果遵循动态链接，在打包镜像时还需要复制若干动态链接库文件，为了简化演示过程，这里使用了静态链接）

```shell
gcc --static -O2 -o say ./say.c
```

```shell
❯ du -h ./say
904K	./say
```

```shell
./say
hello #输入
say:hello
```

接下来编写**Dockerfile**文件，

```dockerfile
FROM scratch
WORKDIR /
COPY ./say /
CMD ["/say"]
```

为了压缩镜像体积，这里空白镜像`scratch`作为基础镜像，并且镜像中只拷贝了`say`可执行文件。（当然，生产环境下不可能如此精简。）

然后执行构建命令。

```shell
docker build -t say .
```

执行完毕后，`docker images`可以看到**REPOSITORY**为**say**的镜像文件，其大小仅为`923kB`。

```shell
❯ sudo docker images
REPOSITORY               TAG       IMAGE ID       CREATED         SIZE
say                      latest    aa5bb1e797b5   8 seconds ago   923kB
busybox                  latest    cabb9f684f8b   8 days ago      1.24MB
```

最后，从**say**镜像启动一个容器来验证是否成功。

```shell
❯ docker run -it --name say-c say
hello
say:hello
```

从上可知，`say`在容器内正常执行。
