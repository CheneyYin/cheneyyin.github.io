---
layout: post
title: "Docker组件架构"
subtitle: "介绍docker架构中的常见组成部分"
date: 2022-01-17
author: "Cheney.Yin"
header-img: "img/bg-material.jpg"
tags:
 - Docker
 - dockerd
 - docker-init
 - docker-proxy
 - containerd
 - containerd-shim
 - ctr
 - runc
---

# Docker组件架构

## 总概

| 组件类别       | 组件            | 介绍                                                         |
| -------------- | --------------- | ------------------------------------------------------------ |
| docker相关组件 | docker          | **docker**客户端，负责发送**docker**操作请求。               |
| 同上           | dockerd         | **docker**服务入口，负责接受客户端请求并返回请求结果。       |
| 同上           | docker-init     | 当业务主进程没有回收能力时，docker-init可以作为容器的1号进程，负责管理容器内的子进程。 |
| 同上           | docker-proxy    | **docker**的网络实现，通过设置`iptables`规则使得访问主机的流量可以发送至容器内。 |
| containerd相关 | containerd      | 负责管理容器的生命周期， 通过接收dockerd的请求，执行启动或者销毁容器。 |
| 同上           | contianerd-shim | 将containerd和真正的容器进程解耦。使用containerd-shim作为容器的父进程，可以实现重启containerd而不影响已经启动的容器进程。 |
| 同上           | ctr             | containerd的客户端， 可以向containerd发送容器请求，主要用来调试开发。 |
| 容器运行时组件 | runc            | 命令行工具，通过调用**namespace**、**cgroup**等系统接口，实现容器的创建和销毁。 |

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/resources/docker-arch.svg" width = "75%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      图1 docker组建架构
  	</div>
</center>
