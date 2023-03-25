---
layout: post
title: "Vert.x core 2 --- Verticles"
subtitle: "介绍Verticles的相关概念和使用"
date: 2022-08-05
author: "Cheney.Yin"
header-img: "//imgloc.com/i/iADI63"
tags:
 - Vert.x
 - vertx-core
 - Verticles
 - 计数器
---

# Verticles

Vert.x提供的Verticles是一种简单便捷、可扩展的、类似Actor Model(非严格Actor Model)的部署和并发模型机制。

Verticle是可选的，不强制使用这种方式创建应用程序。

## 编写Verticle

Verticle的实现类必须实现Verticle接口。但通常直接从抽象类`AbstractVerticle`继承更简单。

```java
public class MyVerticle extends AbstractVerticle {
    //Vert.x部署Verticle时调用
    public void start() {
        
    }
    
    //可选 -- Vert.x撤销Verticle时调用
    public void stop() {
        
    }
}
```

## 异步启动和停止

```java
public class MainVerticle extends AbstractVerticle {
  private HttpServer httpServer;
  @Override
  public void start(Promise<Void> startPromise) throws Exception {
    this.httpServer = vertx.createHttpServer();
    this.httpServer.requestHandler(req -> {
      req.response()
        .putHeader("content-type", "text/plain")
        .end("Hello from Vert.x!");
    }).listen(8888, http -> {
      if (http.succeeded()) {
        startPromise.complete();
        System.out.println("HTTP server started on port 8888");
      } else {
        startPromise.fail(http.cause());
      }
    });
  }

  @Override
  public void stop(Promise<Void> stopPromise) throws Exception {
    this.httpServer.close(ar -> {
      if (ar.succeeded()) {
        System.out.println("Close http server.");
        stopPromise.complete();
      } else {
        System.out.println("Fail to close http server.");
        stopPromise.fail(ar.cause());
      }
    });
  }

  public static void main(String[] args) {
    Vertx vertx = Vertx.vertx();
    MainVerticle verticle = new MainVerticle();
    Future<String> deployIDFuture = vertx.deployVerticle(verticle);
    deployIDFuture.onComplete(ar -> {
      String id = ar.result();
      System.out.println("deploy id:" + id);
      vertx.undeploy(id, cHandler -> {
        if (cHandler.succeeded()) {
          System.out.println("Undeploy Succeeded.");
        } else {
          System.out.println("Undeploy Failed.");
        }
      });
    });

  }
}
```

输出如下：

```shell
HTTP server started on port 8888
deploy id:09073503-56f3-473a-9970-50fd5bfa4d64
Close http server.
Undeploy Succeeded.
```

## Standard verticles和Worker verticles

Standard Verticle被创建时，它会被分派一个Event Loop线程，并在这个Event Loop中执行它的`start`方法。当你在一个Event Loop上调用Core API中的方法并传入处理器时，Vert.x将保证用与调用该方法时相同的Event Loop来执行这些处理器。

- 这意味着我们可以保证Verticle实例中所有的代码都在相同的Event Loop中执行。
- 可以使用单线程的方式编写代码，让Vert.x考虑线程和扩展问题。避免了传统多线程应用经常遇到的竞态条件和死锁问题。

Worker Verticle和Standard Verticle很像，但它并不是由一个Event Loop来执行，而是由Vert.x中的Worker Pool中线程执行。

- Worker Verticle设计用于调用阻塞代码，它不会阻塞任何Event Loop。

示例如下：

```java
public void start(Promise<Void> startPromise) throws Exception {
    this.httpServer = vertx.createHttpServer();
    this.httpServer.requestHandler(req -> {
      req.response().putHeader("content-type", "text/plain").end("Thread ID:" + Thread.currentThread().getName());
      System.out.println((counter++) + "|Thread ID:" + Thread.currentThread().getName());
    }).listen(8888, http -> {
      if (http.succeeded()) {
        startPromise.complete();
        System.out.println("HTTP server started on port 8888");
      } else {
        startPromise.fail(http.cause());
      }
    });
}
    ... ...
public static void main(String[] args) {
    Vertx vertx = Vertx.vertx();
    Verticle verticle = new MainVerticle();
    DeploymentOptions deploymentOptions = new DeploymentOptions().setWorker(false);
    Future<String> deployIDFuture = vertx.deployVerticle(verticle, deploymentOptions);
    deployIDFuture.onComplete(ar -> {
      String id = ar.result();
      System.out.println("deploy id:" + id);
    });
}
```

不断请求`http://localhost:8888`，输出如下：

```shell
HTTP server started on port 8888
deploy id:4b585c77-3f58-4f98-b3fc-6f4030f53fe7
0|Thread ID:vert.x-eventloop-thread-0
1|Thread ID:vert.x-eventloop-thread-0
2|Thread ID:vert.x-eventloop-thread-0
3|Thread ID:vert.x-eventloop-thread-0
4|Thread ID:vert.x-eventloop-thread-0
```

可以发现，所有处理都是在`vert.x-eventloop-thread-0`线程上进行的。

通过如下设置，将Verticle配置为Worker Verticle，

```java
DeploymentOptions deploymentOptions = new DeploymentOptions().setWorker(true);
```

再次请求`http://localhost:8888`，输出如下：

```shell
HTTP server started on port 8888
deploy id:40b8024e-745d-4541-995c-7c8bdbf53d3c
0|Thread ID:vert.x-worker-thread-1
1|Thread ID:vert.x-worker-thread-2
2|Thread ID:vert.x-worker-thread-3
3|Thread ID:vert.x-worker-thread-4
4|Thread ID:vert.x-worker-thread-5
```

可以发现，每次处理并不是在同一个线程上进行的。

## 编程方式部署Verticle

使用`deployVerticle`方法部署Verticle，可以传入Verticle名称或者Verticle实例。

> 通过Verticle实例来部署Verticle仅限Java语言

如下示例使用Verticle实例部署：

```java
Verticle myVerticle = new MyVerticle();
vertx.deployVerticle(myVerticle);
```

使用Verticle名称部署，Verticle的名称回用于查找实例化Verticle的特定`VerticleFactory`。不同的`VerticleFactory`可用于实例化不同语言的Verticle，也可以用于其它目的。因此可以部署Vert.x支持的任何语言编写的Verticle。

示例如下：

```java
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle");

// 部署JavaScript的Verticle
vertx.deployVerticle("verticles/myverticle.js");

// 部署Ruby的Verticle
vertx.deployVerticle("verticles/my_verticle.rb");
```

### Verticle名称到Factory的映射规则

- Verticle名称前可增加一个以`:`结尾的**前缀**，这个前缀用于查找Factory，

  ```shell
  js:foo.js // 使用JavaScript的Factory
  groovy:com.mycompany.SomeGroovyCompiledVerticle // 用Groovy的Factory
  service:com.mycompany:myorderservice // 用Service的Factory
  ```

- 不指定前缀，Vert.x将根据Verticle名称的**后缀**来查找对应Factory，

  ```shell
  foo.js // 将使用JavaScript的Factory
  SomeScript.groovy // 将使用Groovy的Factory
  ```

- 前后缀都未指定，Vert.x将假定Verticle名称是一个Java全限定类名(FQCN)，并尝试实例化它。

### 如何定位Verticle Factory

大部分Verticle Factory会从classpath中加载，并在Vert.x启动时注册。

也可以使用编程的方式去注册或注销Verticle Factory：通过`registerVerticleFactory`方法和`unregisterVerticleFactory`方法。

### 撤销部署

使用`undeploy`方法撤销部署好的Verticle。

```java
vertx.undeploy(deploymentID, res -> {
  if (res.succeeded()) {
    System.out.println("Undeployed ok");
  } else {
    System.out.println("Undeploy failed!");
  }
});
```

### 设置Verticle实例数量

```java
DeploymentOptions options = new DeploymentOptions().setInstances(2);
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle", options);
```

### 向Verticle传入配置

通过在部署时传给Verticle一个JSON格式的配置

```java
JsonObject config = new JsonObject().put("name", "tim").put("directory", "/blah");
DeploymentOptions options = new DeploymentOptions().setConfig(config);
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle", options);
```

传入后，这个配置可以通过Verticle的Context对象访问或者使用`config`方法访问。

```java
System.out.println(this.config.getString("name"));
```

## 从命令行运行Verticle

在命令行中使用`vertx`运行Verticle，`vertx`可使用`sdk`工具安装。

> 1. 安装sdkman
>
>    ```shell
>    curl -s "https://get.sdkman.io" | bash  
>    ```
>    打开新的shell执行`sdk version`验证安装是否成功。
>    
> 2. 使用sdkman安装vert.x命令行工具
>
>    ```shell
>    sdk install vertx
>    ```
>

通过如下方式启动，

```java
vertx run VerticleClass
```

通过灵活配置环境变量`CLASSPATH`来，使得`VerticleClass`可以被搜索到。(例如配合Maven使用)

## Context对象

- Event Loop Context

  > 一般，context会是一个Event Loop Context，它绑定到一个特定的Event Loop线程上。在该context上执行的操作总是在同一个Event Loop线程中。

- Worker Context

  > 运行阻塞代码的Worker Verticle，会关联一个Worker Context，并且所有的操作都会运行在Worker线程池的线程上。

> 一个Context对应一个Event Loop(或Worker)线程，但一个Event Loop线程可能对应多个Context。

## 执行周期性/延迟性操作

由于在Standard Verticle中让线程休眠来引入延迟，会阻塞Event Loop线程；所以取而代之的是使用Vert.x定时器。

### 一次性计时器

一次性计时器会在一定延迟后调用一个Event Handler，以毫秒为单位计时

```java
long timerID = vertx.setTimer(1000, id -> {
   System.out.println("And one second later this is printed.") ;
});
System.out.println("First this is printed.");
```

返回的`timerID`可用于取消计时器。

### 周期性计时器

```java
Vertx vertx = Vertx.vertx(); 
long timerID = vertx.setPeriodic(1000l, id -> {       
  System.out.println(System.currentTimeMillis() + "|Periodic exec."); 
}); 
System.out.println(System.currentTimeMillis() + "|Main exec.");       
```

第一次触发前有一段时间延迟。该计时器会定时触发。

> 如果定时任务会花费大量时间，则计时器事件可能会连续执行，甚至重叠执行。可以考虑使用`setTimer`方法来替代。

```java
class TimerHandlerImpl implements Handler<Long> {
  private Vertx vertx;

  public TimerHandlerImpl(Vertx vertx) {
    this.vertx = vertx;
  }

  @Override
  public void handle(Long event) {
    System.out.println(System.currentTimeMillis() + "|Timer exec.");
    vertx.setTimer(1000l, this);
  }
}

public class TimerDemo {
  public static void main(String[] args) {
    Vertx vertx = Vertx.vertx();
    Handler<Long> timerHandler = new TimerHandlerImpl(vertx);
    long timerID = vertx.setTimer(1000l, timerHandler);
    System.out.println(System.currentTimeMillis() + "|Main exec.");
  }
}
```

### 取消计时器

```java
vertx.cancelTimer(timerID);
```

> 当Verticle被撤销时，计数器会被自动关闭。
