---
layout: post
title: "Vert.x core 1 --- Fluent API"
subtitle: "介绍Vert.x流式Api的概念和使用"
date: 2022-08-01
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags:
    - Vert.x
    - vertx-core
    - Fluent API
    - Reactor
    - Future
---

# 流式(Fluent)API

```java
request.response()
    	.putHeader("Content-Type", "text/plain")
    	.end("Some Text");
```

一个流式 API 表示将多个方法的调用链接在一起。这是 Vert.x API 的通用模式。

## Don't call us, we'll call you.

Vert.x 的 API 大部分是**事件驱动**的。这意味着当关注的事件发生时，Vert.x API 会调用*处理器*处理事件。例如：

-   触发一个计时器
-   Socket 收到一些数据
-   从磁盘读取一些数据
-   发生异常
-   HTTP 服务器收到一个请求

例如每隔一秒发送一个事件的计时器：

```java
vertx.setPeriodic(1000, id -> {
    System.out.println("timer fired.");
});
```

例如收到一个 HTTP 请求：

```java
server.requestHandler(request -> {
    request.response().end("Hello World.");
});
```

## 不要阻塞我

Vert.x 中几乎所有的 API 都不会阻塞调用线程，除了个别特例(如以`Sync`结尾的某些文件系统操作)。

可以立即提供结果的 API 会立即返回，否则会提供一个处理器(handler)来接收稍后回调的事件。

> 如果是调用阻塞 API，在结果返回前调用线程会被阻塞，等待结果不做任何事情。为了防止应用程序停止运转，则需要创建大量线程。然而，这些线程使用内存和线程上下文切换是很可观的。所以，阻塞式的方式对于现代应用所需要的并发级别来说难于扩展。

## Reactor 模式和 Multi-Reactor 模式

一个标准的 Reactor 实现中，有一个独立的 Event Loop 会循环执行，处理所有到达事件，并传递给处理器处理。

单一线程的问题在于它在任意时刻只能运行在一个 CPU core 上。如果希望单线程 Reactor 扩展到多核服务器， Vert.x 则非常方便。Vert.x 创建的每个 Vertx 对象实例维护的是多个 Event Loop 线程；默认情况下，会根据机器上可用的 CPU Core 数量来设置 Event Loop 的数量，也可以自行设置。这种模式成为**Multi-Reactor**。

> 即使一个 Vertx 实例维护了多个 Event Loop，任何一个特定的 handler 用于不会被并发执行。大部分情况下（除 Worker Verticle）它们总在同一个 Event Loop 线程中被调用。

## 黄金法则：不要阻塞 Event Loop

尽管 Vert.x 的 API 是非阻塞的，且不会阻塞 Event Loop。但是用户编写的 handler 中可能会阻塞 Event Loop。

**如果 Event Loop 被阻塞，那么该线程将不再做任何事情；如果 Vertx 实例的全部 Event Loop 都被阻塞了，那么该应用将会完全停止！**

常见的阻塞做法：

-   Thread.sleep()
-   等待一个锁
-   等待一个互斥信号或监视器
-   执行一个长时间数据库操作并等待结果
-   执行一个复杂的计算，占用了可感知的时长
-   在循环语句中长时间逗留

为了帮助诊断这种问题，Vert.x 在检测到 Event Loop 有一段时间没有响应，将会自动记录这种警告，并产生警告日志。比如：

```shell
Thread vertx-eventloop-thread-3 has been blocked for 20458 ms
```

Vert.x 还将提供堆栈跟踪，来精准定位发生阻塞的位置。

> 如果想关闭这些警告或配置，可以在 VertxOptions 中设置。

## Future 的异步结果

Vert.x 4 使用 future 承载异步结果。异步方法会返回一个`Future`对象，其中包含成功或者失败的异步结果。当 future 执行完成后，结果可用时会调用 handler 进行处理。

```java
Vertx vertx = Vertx.vertx();
FileSystem fs = vertx.fileSystem();
Future<FileProps> result = fs.props("./README.adoc");
result.onComplete( (AsyncResult<FileProps> asyncResult) -> {
  if (asyncResult.succeeded()) {
    System.out.println(asyncResult.result().creationTime());
  } else {
    System.out.println(asyncResult.cause());
  }
});
```

## Future 组合

`compose`方法用于顺序组合 future：

-   当前 future 成功，执行 compose 方法指定的方法，该方法会返回新的 future；当返回新的 future 完成时，future 组合成功；
-   当前 future 失败，则 future 组合失败。

```java
Vertx vertx = Vertx.vertx();
FileSystem fs = vertx.fileSystem();
fs.createFile("a.txt")
  .compose(v -> { return fs.writeFile("a.txt", Buffer.buffer("hello")); })
  .compose(v -> { return fs.move("a.txt", "b.txt"); })
  .onComplete(r -> {
    if (r.succeeded()) {
      System.out.println("Succeeded.");
    } else {
      System.out.println("Failed.");
    }
  });
```

示例中，三个操作被串行：

1. 创建一个文件
2. 向文件写入`hello`
3. 移动文件

以上三步，任何异步失败，最终 Future 就是失败。

## Future 协作

Vert.x 中的 future 支持协调多个 Future，支持并发组合和顺序组合。

-   `CompositeFuture.all`方法接受多个 Future 对象作为参数，当所有的 Future 都成功完成时，该方法将返回一个成功的 Future；当任何一个 Future 执行失败，则返回一个失败的 Future：

    ```java
    HttpServer httpServer = vertx.createHttpServer();
    httpServer.requestHandler(request -> {});
    Future<HttpServer> httpServerFuture = httpServer.listen(8888);

    NetServer netServer = vertx.createNetServer();
    netServer.connectHandler(socket -> {});
    Future<NetServer> netServerFuture = netServer.listen(9999);

    CompositeFuture.all(httpServerFuture, netServerFuture)
                    .onComplete(ar -> {
                      if (ar.succeeded()) {
                        System.out.println("Succeeded.");
                      } else {
                        System.out.println("Failed.");
                      }
                    });
    ```

-   `CompositeFuture.any`方法接受多个 Future 作为参数，当任意一个 Future 成功得到结果，则该 Future 成功；当所有的 Future 都执行失败，则该 Future 失败。

    ```java
    FileSystem fs = vertx.fileSystem();
    Future<Void> createFuture0 = fs.createFile("t.txt");
    Future<Void> createFuture1 = fs.createFile("t.txt");
    CompositeFuture.any(createFuture0, createFuture1)
                  .onComplete(ar -> {
                    if (ar.succeeded()) {
                      System.out.println("Succeeded.");
                    } else {
                      System.out.println("Failed.");
                       }
                       fs.delete("t.txt");
                });
    ```

-   `CompositeFuture.join`方法接受多个 Future 作为参数，并将结果合并成一个 Future。该方法会**等待所有 Future 完成，无论成败**。当全部 Future 成功执行完成，得到的 Future 是成功状态的；当至少一个 Future 执行失败时，得到的 Future 失败的。

    ```java
    FileSystem fs = vertx.fileSystem();
    Future<Void> createFuture0 = fs.createFile("t.txt").compose((v)->{
      System.out.println("Exec create 0.");
      return Future.succeededFuture();
    });
    Future<Void> createFuture1 = fs.createFile("s.txt").compose((v)->{
      System.out.println("Exec create 1.");
      return Future.succeededFuture();
    });
    Future<Object> failedFuture0 = Future.failedFuture("Fail").onFailure((v)->{
      System.out.println("Exec failed 0.");
    });
    CompositeFuture.join(failedFuture0, createFuture0, createFuture1)
      .onComplete(ar -> {
        System.out.println("Complete.");
        fs.delete("t.txt");
        fs.delete("s.txt");
      });
    ```

    输出结果为：

    ```shell
    Exec failed 0.
    Exec create 0.
    Exec create 1.
    Complete.
    ```

    `failedFuture0`失败后，其它任务仍然执行，最后输出`complete`。如果，这里把`join`替换为`all`，可能输出的结果如下：

    ```shell
    Exec failed 0.
    Exec create 0.
    Complete.
    Exec create 1.
    ```

    在`createFuture1`未完成前，`all`检测到`Failure`直接调用`onComplete`，没有等待全部完成。

## 兼容 CompletionStage

Vert.x 的 Future API 可兼容 CompletionStage，使用`toCompletionStage`方法将 Vert.x 的 Future 对象转为 CompletionStage 对象，如：

```java
String d = "vertx.io";
Future<String> future = vertx.createDnsClient().lookup(d);
future.toCompletionStage().whenComplete((ip, err) ->{
  if (err != null) {
    System.out.println(d + ":" + ip);
  } else {
    System.out.println("Could not resolve " + d);
    err.printStackTrace();
});
```

相应的，可以使用`Future.fromCompletionStage`方法将 CompletionStage 对象转为 Vert.x 的 Future 对象。

```java
CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(()->{
  System.out.println("Exec stage0.");
  return "Exec stage0.";
});

Future.fromCompletionStage(completableFuture, vertx.getOrCreateContext())
   .onComplete(ar -> {
        if (ar.succeeded()) {
        System.out.println("Succeeded.");
    } else {
        System.out.println("Failed.");
    }
   });
```
