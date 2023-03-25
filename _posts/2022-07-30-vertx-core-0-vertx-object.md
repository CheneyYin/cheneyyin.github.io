---
layout: post
title: "Vert.x core 0 --- Vertx对象"
subtitle: "vertx-core配置使用"
date: 2022-07-30
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags:
 - Vert.x
 - vertx-core
---

# Vert.x Core

> Vert.x Core中的功能相当底层，不包含诸如数据库访问、授权和高层Web应用的功能。这些功能见Vert.x ext。

**Vert.x Core**小巧轻便，可以嵌入现存应用中；没有强制要求使用特定的方式构造应用；支持多种语言编写(JavaScript、Ruby等)。

通过如下maven配置，可在项目中加入**Vert.x Core**依赖。

```xml
<dependency>
 <groupId>io.vertx</groupId>
 <artifactId>vertx-core</artifactId>
 <version>4.2.7</version>
</dependency>
```

## 多线程

- 单个进程内创建多个线程，线程间共享内存。

- 线程内阻塞IO操作使得线程被挂起；创建线程很多时(处理

  [C10K问题]: https://en.wikipedia.org/wiki/C10k_problem

  )，OS调度延迟增加；另外，每次创建新线程大约消耗1MB内存。


## Vertx对象

`Vertx`对象是Vert.x的控制中心，负责几乎一切事情的基础，如创建客户端、服务器、获取时间总线的引用、设置定时器等等。

```java
Vertx vertx = Vertx.vertx();
```

> 大部分应用只需要一个Vert.x实例，如果需要也可以创建多个Vert.x实例，如：隔离的事件总线或不同组的客户端和服务器。

### 创建Vertx对象时指定配置项

```java
Vertx vertx = Vertx.vertx(new VertxOptions().setWorkerPoolSize(40));
```

`VertxOptions`对象有很多配置，包括集群、高可用、池大小等。

### 创建集群模式的Vertx对象

创建集群模式`Vertx`对象一般需要异步方式来创建。

> 让不同的Vertx实例组成一个集群需要一些时间，Vert.x在这段时间内没有阻塞调用线程，会以异步方式返回结果。

示例如下：

```java
VertxOptions options = new VertxOptions();
Vertx.clusterVertx(options, res -> {
    if (res.succeeded()) {
        Vertx vertx = res.result();
    } else {
        ...
    }
});
```