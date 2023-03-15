---
layout: post
title: "简易Docker仓库"
subtitle: "介绍常用的docker镜像仓库和使用方法"
date: 2022-05-30
author: "Cheney.Yin"
header-img: "img/bg-material.jpg"
tags:
 - Docker
 - docker registry
---

# 简易docker仓库

docker.io提供了简易的docker仓库实现，可以从docker.hub上拉取。

```shell
docker pull registry
```

registry支持本地模式和远程模式。其中，本地模式仅限于本地访问，无需配置网络服务地址。

## 1 本地模式

本地模式启动方式如下，

```shell
docker run -d -v Your_local_registry:/var/lib/registry/ -p 5000:5000 --name registry registry
```

如果，本地存在镜像`localhost:5000/say:latest`，可以直接使用`docker push`命令推送之`registry`服务。

```shell
docker push localhost:5000/say:latest
```

假使，本地删除了该镜像，还是可以再次从`registry`服务拉取镜像。

```shell
docker rmi localhost:5000/say:latest
docker images
REPOSITORY       TAG       IMAGE ID       CREATED          SIZE

docker pull localhost:5000/say:latest
docker images
REPOSITORY       TAG       IMAGE ID       CREATED          SIZE
localhost:50000/say   latest    8b196f4277c4   12 minutes ago   923kB
```

## 2 远程模式

在远程模式下，镜像的存取可以通过网络跨主机进行。在远程模式下，需要为`registry`服务配置`https`所需要的证书、密钥、网络地址等。

`cfssl`、`openssl`等工具可以方便地签发证书，[相关操作可以参考cfssl.md](./cfssl.md)。

（1）证书

建立**CA**证书签发请求文件和**CA**配置文件，

```shell
./ca-csr.json
{
    "CN": "cheney.io",
    "hosts": [
        "cheney.io"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "JS",
            "L": "Wu Xi"
        }
    ]
}

./ca.config
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "server": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
			"peer": {
				"expiry": "43800h",
				"usages": [
					"signing",
					"key encipherment",
					"server auth",
					"client auth"
				]
			}
        }
    }
}
```

创建**CA**自签证书，的证书`ca.pem`、密钥`ca-key.pem`和签发请求`ca.csr`。

```shell
cfssl gencert --initca ./ca-csr.json | cfssljson -bare ca
```

将`ca.pem`转为`ca.crt`格式。

```shell
openssl x509 --outform der -in ./ca.pem -out ./ca.crt
```

`ca.crt`证书需要安装在参与认证的设备上。

```shell
sudo cp ./ca.crt /etc/ca-certificates/trust-source/anchors/ca.crt
sudo update-ca-trust
```

**CA证书更新仅对执行命令的进程和新的进程生效。**

接下来，为`registry`签发证书。

首先，定义签发请求，

```shell
./server-csr.json
{
    "CN": "registry.cheney.io",
    "hosts": [
        "172.17.0.1",
		"172.17.0.2",
		"172.17.0.3",
		"172.17.0.4"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "JS",
            "L": "Wu Xi"
        }
    ]
}
```

```shell
cfssl gencert -ca ../ca.pem -ca-key ../ca-key.pem --config ../ca.config --profile server ./server-csr.json| cfssljson -bare server
```

得到证书`server.pem`、`server-key.pem`和`server.csr`。

（2）启动容器

```shell
docker run -d \
-v /home/cheney/docker-registry:/var/lib/register \
-v /home/cheney/expr/CA/server:/certs \
-p 443:443 \
-e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/server.pem \
-e REGISTRY_HTTP_TLS_KEY=/certs/server-key.pem  \
--name registry \
registry
```

`/home/cheney/expr/CA/server`目录下放置了`registry`的证书和私钥，需要映射之容器内`/certs`。

（3）连接测试

推送：

```shell
docker push 172.17.0.2/say:latest
```

拉取：

```shell
docker pull 172.17.0.2/say:latest
```



