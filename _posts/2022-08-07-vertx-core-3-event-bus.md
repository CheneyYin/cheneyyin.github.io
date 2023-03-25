---
layout: post
title: "Vert.x core 3 --- Event Bus"
subtitle: "介绍Event Bus相关概念和使用"
date: 2022-08-07
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags:
 - Vert.x
 - vertx-core
 - Event Bus
 - 消息发布/订阅
---

# Event Bus

- Event Bus构建了一个跨越**多个服务器节点**和**多个浏览器**的分布式**点对点消息系统**。
- Event Bus支持发布/订阅、点对点、请求-响应的消息传递方式。
- Event Bus的API很简单。基本只涉及注册/注销handler、发布和发送消息。

## 基本概念

- 寻址

  消息的发送目标称为地址。*Vert.x的地址是简单的字符串，任何字符串都合法*。不过建议使用某种规范来进行地址命名。

- 处理器

  消息由处理器接收，处理器需要注册在某个地址上。

  > - 同一地址可以注册多个不同的处理器。
  >
  > - 一个处理器可以注册在多个不同的地址上。

- 发布/订阅消息

  消息发布到一个地址上，意味着信息会被传递给所有注册在改地址上的处理器。

- 点对点消息传递 与 请求-响应消息传递

  - 点对点

    消息发送在一个地址上，Vert.x仅会把消息发给注册在地址上的处理器中的**一个**。如果注册处理器不止一个，则使用轮询选择其中一个。

  - 请求-响应

    当接收者收到消息并且处理完成时，可以选择性的回复消息。若回复，则发送消息时指定的应答处理器将会被调用。

    当发送者收到应答消息时，发送者还可以继续回复这个应答，这个过程可以不断重复。(通过这种方式可以在两个不同的Verticle之间建立一个对话窗口)

- 尽力传输

  Vert.x会尽力传递消息，不会主动丢弃消息。(Best-effort delivery)

  当Event Bus出现故障时，消息可能会丢失。

  为了避免消息丢失，一般发送者会在故障恢复后重试，接收者处理器应当**具备幂等性**，避免重复接收消息的影响。

  > RPC通信语义：at least once、at most once和exactly once。

- 消息类型

  Vert.x默认允许任何基本/简单类型、String类型、buffers类型的值作为消息发送。

  Vert.x建议使用更加规范、通用的JSON格式发送消息。

  Event Bus非常灵活，可以自定义`codec`来实现任何类型的对象在Event Bus上传输。

## Event Bus API

### 注册处理器

- 最简单的注册处理器的方式是使用`consumer`方法，当消息到达处理器时，消息会被放入`message`参数，供处理器调用。
- 也可以使用`consumer`方法返回的`MessageConsumer`对象，通过该对象的`handler`方法配置处理器。
- 在向**集群模式**下的Event Bus注册处理器时，注册信息会花费一些时间才能传播到集群中所有节点，可以在`MessageConsumer`对象上注册一个`CompletionHandler`接收完成注册的通知。

```java
Vertx vertx = Vertx.vertx();
EventBus eventBus = vertx.eventBus();
eventBus.consumer("slot#0", msg -> {
  System.out.println("slot#0-Receive " + msg.body());
});

MessageConsumer<String> consumer = eventBus.consumer("slot#0");
consumer.handler(msg -> {
  System.out.println("consumer slot#0-Receive " + msg.body()); 
  msg.reply("ack");
});

consumer.completionHandler(ar -> {
  if (ar.succeeded()) {
    System.out.println("Register handler succeeded.");
  } else {
    System.out.println("Fail to register handler.");
  }                  
});     

eventBus.publish("slot#0", "hello");

MessageProducer<String> producer = eventBus.publisher("slot#0");
producer.write("ok");
producer.write("do", ar -> {                 
  if (ar.succeeded()) {                
    System.out.println("Send has been received.");     
  } else {                  
    System.out.println("Fail to send.");               
  }                  
});                 
```

### 注销处理器

直接使用`MessageConsumer`对象的`unregister`方法来注销处理器。同时，也可以在方法中配置`CompletionHandler`，接收完成通知。

```java
consumer.unregister(ar -> {                     
  if (ar.succeeded()) {                         
    System.out.println("Unregister succeeded.");
  } else {                                      
    System.out.println("Fail to unregister.");  
  }                                             
});                                             
```

### 发布消息

使用`publish`方法指定一个地址发布即可。**所有**注册在改地址上的处理器**都**可以接收到消息。

```java
eventBus.publish("slot#0", "hello");
```

### 发送消息

使用`send`方法发送消息。**所有**注册在该地址上的处理器，**仅一个**能够接收到消息。

```java
eventBus.send("slot#0", "send hello.");
```

### 设置消息头

在Event Bus上发送消息可包含头信息。在发送或发布时提供一个`DeliveryOptions`来指定头信息。

```java
DeliveryOptions deliveryOptions = new DeliveryOptions();
deliveryOptions.addHeader("X-id", "0x1111"); 
deliveryOptions.addHeader("X-name", "jack");
eventBus.publish("slot#1", "call", deliveryOptions); 
```

### 消息顺序

> Vert.x会按照发送者**发送消息的顺序**，将消息以**同样的顺序**传递给处理器。

### 消息对象

- 消息处理器中接收到的对象类型时`Messaage`。
- 消息的`body`对应发送或发布的对象。
- 消息的头信息可以通过`headers`方法获取。

### 应答消息/发送消息

当使用`send`方法发送消息时，Event Bus会尝试将消息传递到注册在Event Bus上的`MessageConsumer`中。某些情况下，发送者可以通过**请求/响应+模式**来得知消费者已经接收并处理了该消息。

消费者可以调用`reply`方法来应答这个消息，确认该消息已被处理。应答消息会被返回给发送者的应答处理器。

```java
Vertx vertx = Vertx.vertx();
EventBus eventBus = vertx.eventBus();
eventBus.consumer("slot#0", msg -> {
  System.out.println("Headers: " + msg.headers());
  System.out.println("Body:" + msg.body());
  String seqID = msg.headers().get("Seq");
  msg.reply("ACK " + seqID);
}); 

DeliveryOptions deliveryOptions = new DeliveryOptions().addHeader("Seq", "#00001");
eventBus.request(
  "slot#0",          
  "hello.",     
  deliveryOptions,
  (AsyncResult<Message<String>> ar) -> {
  if (ar.succeeded()) {
    Message<String> msg = ar.result();
    System.out.println("Reply: " + msg.body());
  }                      
});
```

> `request`方法同`send`方法一样，是**点对点**。

### 超时发送

可以在`DeliveryOptions`中指定超时时间，如果在这个时间内没有收到应答，则会以**失败结果**为参数调用应答处理器。

> 默认超时时间为30秒。

```java
public class DeliveryTimeOutDemo {
  public static void main(String[] args) {
    Vertx vertx = Vertx.vertx();
    EventBus eventBus = vertx.eventBus();
    eventBus.consumer("slot#0", msg -> {
      System.out.println("Headers:" + msg.headers());
      System.out.println("Body:" + msg.body());

      String seqID = msg.headers().get("Seq");
      vertx.setTimer(5000l, id -> {
        msg.reply("ACK " + seqID);
      });
    });

    eventBus.request(
      "slot#0",
      "Hello",
      new DeliveryOptions().addHeader("Seq", "#000001").setSendTimeout(1000l),
      (AsyncResult<Message<String>> ar) -> {
        if (ar.succeeded()) {
          Message<String> msg = ar.result();
          System.out.println("Receive reply: " + msg.body());
        } else {
          System.out.println("Fail to receive reply");
          System.out.println(ar.cause());
        }
      }
      );
  }
}
```

### 发送失败

消息发送失败的原因可能如下：

- 没有可用的处理器接收消息。
- 接收者调用了`fail`方法显式声明失败。

发生这些情况，应答处理器将会以这些异常失败结果为参数进行调用。

### 消息编解码器

在Event Bus中可以发送任何对象，只需要为这个对象类型注册一个编解码即可。

每个消息编解码器都有一个名称，需要在发送或者发布消息时通过`DeliveryOptions`来指定：

```java
eventBus.registerCodec(myCodec);
DeliveryOptions options = new DeliveryOptions().setCodecName(myCodec.name());
eventBus.send("orders", new MyPOJO(), options);
```

如果某个特定类总是使用特定的解码器，那么你可以为这个类注册默认编解码器，不需要在每次发送时指定：

```java
eventBus.registerDefaultCodec(MyPOJO.class, myCodec);
eventBus.send("orders", new MyPOJO());
```

具体示例如下：

定义一个POJO类，

```java
class OnePOJO {
  private int id;
  private String content;
  public OnePOJO() {}

  public OnePOJO(int id, String content) {
    this.id = id;
    this.content = content;
  }

  public int getId() {
    return id;
  }

  public void setId(int id) {
    this.id = id;
  }

  public String getContent() {
    return content;
  }

  public void setContent(String content) {
    this.content = content;
  }

  @Override
  public String toString() {
    return "OnePOJO{" +
      "id=" + id +
      ", content='" + content + '\'' +
      '}';
  }
}
```

实现一个编解码器，

```java
class CodecImpl implements MessageCodec<OnePOJO, OnePOJO> {

  @Override
  public void encodeToWire(Buffer buffer, OnePOJO onePOJO) {
    buffer.appendInt(onePOJO.getId())
          .appendInt(onePOJO.getContent().length())
          .appendString(onePOJO.getContent(), "UTF-8");
  }

  @Override
  public OnePOJO decodeFromWire(int pos, Buffer buffer) {
    int id = buffer.getInt(pos);
    int contentLen = buffer.getInt(pos + 4);
    String content = buffer.getString(pos + 8, pos + 8 + contentLen, "UTF-8");
    return new OnePOJO(id, content);
  }

  @Override
  public OnePOJO transform(OnePOJO onePOJO) {
    return onePOJO;
  }

  @Override
  public String name() {
    return "OnePOJO-Codec";
  }

  @Override
  public byte systemCodecID() {
    return -1;
  }
}
```

注册编解码器，

```java
public class CodecDemo {
  public static void main(String[] args) {
    Vertx vertx = Vertx.vertx();
    EventBus eventBus = vertx.eventBus();
    CodecImpl codec = new CodecImpl();
    eventBus.registerCodec(codec);

    eventBus.consumer("slot#0", (Message<OnePOJO> msg) -> {
      System.out.println("Consumer: " + msg.body());
    });

    eventBus.send(
      "slot#0",
      new OnePOJO(1, "test"),
      new DeliveryOptions().setCodecName(codec.name())
      );
  }
}
```

输出如下：

```shell
Consumer: OnePOJO{id=1, content='test'}
```

> 消息编解码器的编码输入和解码输出**可以为不同类型**。例如，可以编写一个编解码器来发送MyPOJO类的对象，但是当消息发送给处理器后解码成MyOtherPOJO对象。

## 集群模式的Event Bus

```java
public class ClusterDemo {
  public static void main(String[] args) {
    VertxOptions vertxOptions = new VertxOptions();
    Vertx.clusteredVertx(vertxOptions, ar -> {
      if (ar.succeeded()) {
        Vertx vertx = ar.result();
        EventBus eventBus = vertx.eventBus();
        System.out.println("Cluster Event Bus has been created.");
      } else {
        System.out.println("Fail to create Cluster Event Bus.");
        System.out.println(ar.cause());
      }
    });
  }
}
```

通过如下命令启动两个进程，

```shell
java -cp ./target/starter-1.0.0-SNAPSHOT-jar-with-dependencies.jar com.cheney.starter.ClusterDemo   
java -cp ./target/starter-1.0.0-SNAPSHOT-jar-with-dependencies.jar com.cheney.starter.ClusterDemo   
```

观察日志发现`Members`，

```shell
Members {size:2, ver:2} [
	Member [192.168.43.34]:5701 - d35d1f70-082b-4359-a584-2cc467308c52
	Member [192.168.43.34]:5702 - 307af634-d635-48b0-92be-dbec53c0bf88 this
]
```

## 配置Event Bus

在`VertxOptions`中配置`EventBusOptions`。

```java
VertxOptions options = new VertxOptions()
    .setEventBusOptions(new EventBusOptions()
        .setClusterPublicHost("whatever")
        .setClusterPublicPort(1234)
    );
```
