---
layout: post
title: "sudo和wheel用户组"
subtitle: "介绍wheel用户组的用途"
date: 2022-06-25
author: "Cheney.Yin"
header-img: "img/bg-material.jpg"
tags:
 - Linux
 - sudo
 - wheel用户组
---

# sudo 和 wheel组

查看`docker`用户信息，

```shell
id docker
uid=1001(docker) gid=1001(docker) groups=1001(docker)
```

`docker`用户没有加入`/etc/sudoers`，因此`sudo`命令失败。

```shell
[docker@cheney-hp etc]$ sudo ip addr
[sudo] password for docker:
docker is not in the sudoers file.  This incident will be reported.
```

解决该问题有两种方式，一种是把`docker`加入`/etc/sudoers`，例如`echo 'docker ALL=(ALL) ALL' >> /etc/sudoers`，另一种方式是把`docker`加入`wheel`用户组。

```shell
sudo gpasswd -a docker wheel
```

```shell
[docker@cheney-hp etc]$ sudo ip addr
[sudo] password for docker:
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
```

