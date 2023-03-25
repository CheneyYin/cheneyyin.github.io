---
layout: post
title: "Spark Streaming"
subtitle: "介绍Spark Streaming的基本概念和使用"
date: 2020-03-21
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags:
 - Spark
 - Spark Streaming
 - 流式计算
 - 接收器
 - RDD
 - 微批处理
---

# Spark Streaming

## 概述

**Spark Streaming**是**Spark Core API**的扩展，它支持弹性的、高吞吐、容错的实时数据流的处理。数据可以通过多种数据源获取，例如**Kafka**、**Flume**、**Kinesis**以及**TCP Sockets**，也可以通过例如`map`、`reduce`、`join`、`window`等高阶函数组成的复杂算法处理。最终，处理后的数据可以输出到文件系统，数据库以及实时仪表盘。另外，可以在数据流上使`用Spark机器学习以及图形处理算法。
**Spark Streaming**把实时接收的数据切分成多个小批量数据，然后交由**Spark**引擎处理并分批的生产结果数据流。
**Spark Streaming**提供了一个高层次的抽象叫做离散流（**discretized stream**）即**DStream**，它代表了一个连续的数据流。**DStream**可以通过来自数据源的输入数据流创建，例如**Kafka**、**Flume**以及**Kinesis**，或者在其它**DStream**上进行高层次的操作创建。在内部，一个**DStream**是通过一系列的**RDD**来表示。

## 简单示例
从**TCP Socket**的数据服务器接收文本数据，并计算单词数。
```shell
nc -lk 12345 < ./data.txt
```

```java
public class App {
	public static void main(String[] args) {
		String sourceService = "192.168.32.61";
		int sourcePort = 12345;
		SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("Spark Streaming");
		JavaStreamingContext ssc = new JavaStreamingContext(conf, Durations.seconds(5));
		ssc.sparkContext().setLogLevel("WARN");
		
		JavaReceiverInputDStream<String> ris = ssc.socketTextStream(sourceService, sourcePort);
		JavaDStream<String> words = ris.flatMap(l -> Arrays.asList(l.split(" ")).iterator());
		JavaPairDStream<String, Long> pairs = words.mapToPair(w -> new Tuple2(w, 1l));
		JavaPairDStream<String, Long> wordCounts = pairs.reduceByKey((s, n) -> s + n);
		
		wordCounts.print();
		ssc.start();
		try {
			ssc.awaitTermination();
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}
}
```

## 基本概念
### 依赖
**maven**依赖如下：

```xml
<dependency>
	<groupId>org.apache.spark</groupId>
	<artifactId>spark-streaming_2.11</artifactId>
	<version>2.2.1</version>
</dependency>
```

**Kafka**、**Flume**、**Kinesis**需要添加相对应的组件。常见组件如下，

| 源      | 组件                             |
| ------- | -------------------------------- |
| Kafka   | spark-streaming-kafka-0-8_2.11   |
| Flume   | spark-streaming-flume_2.11       |
| Kinesis | spark-streaming-kinesis-asl_2.11 |

### 初始化StreamingContext
`StreamingContext`对象是**Spark Streaming**所有功能的主入口点。
一个`StreamingContext`对象可以从`SparkConf`创建。

```java
SparkConf conf = new SparkConf().setMaster(master).setAppName(appName);
JavaStreamingContext ssc = new JavaStreamingContext(conf, Durations.seconds(5));
```

其中`appName`为应用名称，`master`集群模式（**Spark standalone**、**mesos**、**YARN**、**local[\*]**）。`SparkContext`在`StreamingContext`内部已创建，通过`ssc.sparkContext()`访问。

批处理间隔必须根据应用程序和可用集群资源的等待时间要求进行设置。[优化指南](http://cwiki.apachecn.org/pages/viewpage.action?pageId=2887785)

一个`StreamingContext`对象也可以从现有的`SparkContext`对象创建出。
一个**Context**定义后，必须做以下几个方面。

1. 通过创建输入**DStreams**定义输入源。
2. 通过应用转换和输出操作**DStreams**定义流计算。
3. 开始接收数据，并用`streamingContext.start()`处理它。
4. 等待处理被停止（手动停止或因任何错误停止），使用`StreamingContext.awaitTermination()`。
5. 该处理可以使用`StreamingContext.stop()`。

**要点**：

1. 一旦一个**context**已经启动，将不会有新的数据流的计算可以被创建或者添加到它。
2. 一旦一个**context**已经停止，它不会被重新启动。
3. 同一时间内**JVM**中只有一个`StreamingContext`可以被激活。
4. 在`StreamingContext`上的`stop()`同样也停止了`SparkContext`。为了只停止`StreamingContext`，设置`stop()`的可选参数`stopSparkContext`为`false`。
5. 一个`SparkContext`就可以被重用以创建多个`StreamingContext`，只要前一个`StreamingContext`在下一个`StreamingContext`被创建之前停止（不停止`SparkContext`）。

### 离散化流

**DStream**是**Spark Streaming**提供的基本抽象，在**Spark**内部，**DStream**由一系列的**RDD**组成，每个**RDD**包含了一定时间间隔的数据。
应用于**DStream**的任何操作会转化为对于底层的**RDD**的操作。例如上述统计单词个数的例子中，转换一个行成为单词中，`flatMap`操作被应用于行离散流中的每个**RDD**来生成单词离散流的**RDD**。
这些底层的**RDD**变换有**Spark engine**计算。**DStream**操作隐藏了大多数这些细节并为了方便起见，提供给了开发者一个更高级别的接口。

### Input DStreams 和 Receivers

**Input DStream**分为两种，一种需要接收器（**Receiver**），另外一种不需要接收器。上文中**TCP Socket**获取文本流，需要建立接收器；**Kafka**、**Flume**、**Kinesis**等也要建立接收器。

需要注意到是，接收器需要占用一个核来接收数据。除了计算需要的CPU核，还需要分配其它核给接收器。

#### Basic Sources

除了**TCP sockets**，**StreamingContext API**还提供了根据文件作为输入源创建离散流的方法。
- 文件流：用于从文件中读取数据，在任何与**HDFS API**兼容的文件系统中（即，HDFS、S3、NFS等），一个离散流可以通过下面方式创建：
  **Spark Streaming**将监控数据目录，并且处理任何在该目录下创建的文件（不支持嵌套目录）。注意：
  - 文件必须具有相同的数据格式。
  - 文件必须在数据目录中通过原子移动或者重命名它们到这个目录下创建。
  - 一旦移动，这些文件必须不能再更改，因此如果文件被连续地追加，新的数据将不会被读取。
  对于简单的文本文件，还有一个更加简单的方法`streamingContext.textFileStream(dataDirectory)`。并且文件流不需要运行一个接收器。

- **Streams based on Custom Receivers**：离散流可以使用通过自定义的接收器来接收到数据来创建。（[自定义接收器](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)）

- **Queue of RDDs as Stream**（**RDDs队列作为一个流**）：为了使用测试数据测试**Spark Streaming**应用程序，还可以使用**streamingContext.queueStream(queueOfRDDs)**创建一个基于**RDD**队列的离散流，每个进入队列的**RDD**都将被视为**DStream**中的一个批次数据，并且就一个流进行处理。

```java
    JavaStreamingContext ssc = new JavaStreamingContext(conf, Durations.seconds(5));
    ssc.sparkContext().setLogLevel("WARN");
    JavaSparkContext sc = ssc.sparkContext();
    Queue<JavaRDD<String>> rddQueue = (Queue<JavaRDD<String>>) new LinkedList<JavaRDD<String>>();
    File[] dataFiles = new File("/home/yanchengyu/dataset").listFiles();
    for (File dataFile : dataFiles) {
        JavaRDD<String> rdd = sc.textFile("file://" + dataFile.getAbsolutePath());
        rddQueue.offer(rdd);
    }

    ssc.queueStream(rddQueue, true)
    .flatMap(l -> Arrays.asList(l.split(" ")).iterator())
    .mapToPair(w -> new Tuple2<String, Long>(w, 1l))
    .reduceByKey((s, n) -> s + n)
    .print();

    ssc.start();
    try {
        ssc.awaitTermination();
    } catch (InterruptedException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    }
```
#### 接收器的可靠性
如果系统从这些可靠的数据来源接收数据，并且被确认正确地接收数据，它可以确保数据不会因为任何类型的失败而导致数据丢失。
1. Reliable Receiver：当数据被接收并存储在**Spark**中并带有备份副本时，一个可靠的接收器（**reliable receiver**）正确地发送确认给一个可靠的数据源（**reliable source**）。
2. Unreliable Receiver：一个不可靠的接收器不发送确认到数据源。这可以用于不支持确认的数据源，或者甚至是可靠的数据源当你不想或者不需要进行复杂的确认的时候。

```java
public class JavaCustomReceiver extends Receiver<String>{
	private String host;
	private int port;
	private Thread daemon;
	
	public JavaCustomReceiver(StorageLevel storageLevel, String host, int port) {
		super(storageLevel);
		// TODO Auto-generated constructor stub
		this.host = host;
		this.port = port;
	}

	@Override
	public void onStart() {
		// TODO Auto-generated method stub
		if (this.daemon == null) {
			this.daemon = new Thread(this::receive);
		}
		this.daemon.start();
	}

	private void receive() {
		try (
			Socket socket = new Socket(host, port);
			BufferedReader reader = new BufferedReader( 
						new InputStreamReader(socket.getInputStream()));
				){
			String line = null;
			while(!this.isStopped() && (line = reader.readLine()) != null) {
				this.store(line.toUpperCase());
			}			
			this.restart("Cannot continue to receive");
		} catch (UnknownHostException e) {
			this.restart("Unknown host", e);
		} catch (IOException e) {
			this.restart("Catch IO exception");
		}
	}

	@Override
	public void onStop() {
		this.daemon.interrupt();
	}

}
```
### DStreams的Transformation操作

#### updateStateByKey

```java
JavaStreamingContext ssc = new JavaStreamingContext(conf, Durations.seconds(5));
    ssc.sparkContext().setLogLevel("WARN");
    ssc.checkpoint("file:///home/yanchengyu/checkpoint");
    String sourceService = "192.168.32.61";
    int sourcePort = 12345;

    JavaReceiverInputDStream<String> ris = ssc.socketTextStream(sourceService, sourcePort);
    JavaDStream<String> words = ris.flatMap(l -> Arrays.asList(l.split(" ")).iterator());
    JavaPairDStream<String, Long> pairs = words.mapToPair(w -> new Tuple2(w, 1l));

    Function2<List<Long>, Optional<Long>, Optional<Long>> func = new Function2<List<Long>, Optional<Long>, Optional<Long>>() {

        @Override
        public Optional<Long> call(List<Long> v1, Optional<Long> v2) throws Exception {
            // TODO Auto-generated method stub
            long sum = v2.orElse(0l);
            sum += v1.stream().mapToLong(v -> v).sum();
            return Optional.of(sum);
        }

    };
    JavaPairDStream<String, Long> wordCounts = pairs.<Long>updateStateByKey(func);
    wordCounts.print();
    ssc.start();
    try {
        ssc.awaitTermination();
    } catch (InterruptedException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    }
```

注意：`Optional`是`org.apache.spark.api.java`下的类。

#### transform操作

**tansform操作**允许在**DStream**运行任何**RDD-to-RDD**函数。它能够被用来应用任何没有在**DStream API**中提供的**RDD**操作。例如过滤出包含某个RDD中关键字的单词。

```java
JavaStreamingContext ssc = new JavaStreamingContext(conf, Durations.seconds(5));
    ssc.sparkContext().setLogLevel("WARN");
    JavaSparkContext sc = ssc.sparkContext();
    JavaRDD<String> filterRdd = sc.textFile("filter.list");
    String sourceService = "192.168.32.61";
    int sourcePort = 12345;
    JavaReceiverInputDStream<String> ris = ssc.socketTextStream(sourceService, sourcePort);
    JavaDStream<String> words = ris.flatMap(l -> Arrays.asList(l.split(" ")).iterator());
    words.transform(rdd -> rdd.intersection(filterRdd)).print();
    ssc.start();
    try {
        ssc.awaitTermination();
    } catch (InterruptedException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    }
```

#### Window操作

窗口在源**DStream**上滑动，合并和操作落入窗内的源**RDD**，产生窗口化的**DStream**的**RDD**。在这个具体的例子中，程序在三个时间单元的数据上进行窗口操作，并且每两个时间单元滑动一次。每个窗口操作都需指定两个参数：

- **window length**：窗口的持续时间。

- **sliding interval**：窗口操作执行时间间隔。
这两个参数必须是源**DStream**上批时间间隔的倍数。
`reduceByKeyAndWindow`有两种用法，第一种用法如下，

```java
//统计窗口内的单词个数
int batchInterval = 5;
JavaStreamingContext ssc = new JavaStreamingContext(conf, Durations.seconds(batchInterval));
ssc.sparkContext().setLogLevel("WARN");
String sourceService = "192.168.32.61";
int sourcePort = 12345;
JavaReceiverInputDStream<String> ris = ssc.socketTextStream(sourceService, sourcePort);
ris.flatMap(l -> Arrays.asList(l.split(" ")).iterator())
.mapToPair(w -> new Tuple2<>(w, 1))
.reduceByKeyAndWindow((s, n) -> s + n, 
        Durations.seconds(batchInterval * 3), 
        Durations.seconds(batchInterval * 2))
.print();
ssc.start();
try {
    ssc.awaitTermination();
} catch (InterruptedException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
}
```

第二种用法，使用`invFunc`处理窗口内的过期数据，更为高效，但需要启用checkpoint，例如

```java 
int batchInterval = 5;
JavaStreamingContext ssc = new JavaStreamingContext(conf, Durations.seconds(batchInterval));
ssc.sparkContext().setLogLevel("WARN");
ssc.checkpoint("file:///home/yanchengyu/checkpoint");
String sourceService = "192.168.32.61";
int sourcePort = 12345;
JavaReceiverInputDStream<String> ris = ssc.socketTextStream(sourceService, sourcePort);
ris.flatMap(l -> Arrays.asList(l.split(" ")).iterator())
.mapToPair(w -> new Tuple2<>(w, 1))
.reduceByKeyAndWindow((s, n) -> s + n,
        (s, n) -> s - n,
        Durations.seconds(batchInterval * 3), 
        Durations.seconds(batchInterval * 2))
.print();
ssc.start();
try {
    ssc.awaitTermination();
} catch (InterruptedException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
}
```

其中，`(s, n) -> s + n`和`(s, n) -> s + n`逆函数。

#### Join操作

```java
JavaSparkContext sc = new JavaSparkContext(conf);
JavaStreamingContext ssc = new JavaStreamingContext(sc, Durations.seconds(5));
sc.setLogLevel("WARN");
String sourceService = "192.168.32.61";
int sourcePort1 = 12345;
JavaDStream<String> rs = ssc.socketTextStream(sourceService, sourcePort1)
.flatMap(l -> Arrays.asList(l.split(" ")).iterator());

rs.mapToPair(w -> new Tuple2<String, String>(Math.random() > 0.5?w.toUpperCase():w, w))
.join(rs.mapToPair(w -> new Tuple2<String, String>(Math.random() > 0.5?w.toUpperCase():w, w))).print();

ssc.start();
try {
    ssc.awaitTermination();
} catch (InterruptedException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
}
```

上述代码对两个**DStream**进行**join**操作。

### DStreams上的输出操作

输出操作允许**DStream**操作推到如数据库、文件系统等外部系统中。因为输出操作实际上是允许外部系统消费转换后的数据，它们出发的实际操作是**DStream**转换。

| 输出操作                              | 含义                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| `print()`                             | 打印前10条元素                                               |
| `saveAsObjectFiles(prefix, [suffix])` | 保存**DStream**的内容为一个序列化文件**SequenceFile**。每个批间隔的文件的文件名基于**prefix**和**suffix**生成。“**prefix-TIME_IN_MS.[suffix]**”。 |
| `saveAsTextFiles(prefix, [suffix])`   | 保存**DStream**的内容为一个文本文件。                        |
| `saveAsHadoopFiles(prefix, [suffix])` | 保存**DStream**的内容为一个**hadoop**文件。                  |
| `foreachRDD(func)`                    | 在从流中生成的每个**RDD**上应用函数**func**的最通用的输出操作。这个函数应该推送每个**RDD**的数据到外部系统。**需要注意**的是**func**函数在驱动程序中执行，并且通常都有**RDD action**在里面推动**RDD**流的计算。 |

#### foreachRDD设计模式使用

**dstream.foreachRDD**是一个强大的原语，发送数据到外部系统中。然而，明白怎样正确地、有效地用这个原语是非常重要的。

```java
dstream.foreachRDD(rdd -> {
  Connection connection = createNewConnection(); // executed at the driver
  rdd.foreach(record -> {
    connection.send(record); // executed at the worker
  });
});
```

上述例子不正确，需要将连接对象序列化后，从**driver**发送到**worker**。连接对象不能传输会表现为序列化错误或者初始化错误。正确地方法应该是在**worker**创建连接对象。

```java
dstream.foreachRDD(rdd -> {
  rdd.foreach(record -> {
    Connection connection = createNewConnection();
    connection.send(record);
    connection.close();
  });
});
```

上述例子会为每个`record`对象建立一个连接对象，开销会很大。

```java
dstream.foreachRDD(rdd -> {
  rdd.foreachPartition(partitionOfRecords -> {
    Connection connection = createNewConnection();
    while (partitionOfRecords.hasNext()) {
      connection.send(partitionOfRecords.next());
    }
    connection.close();
  });
});
```

上述例子，解决了性能问题，每个分区在同一线程内运行，共享同一个连接对象。

```java
dstream.foreachRDD(rdd -> {
  rdd.foreachPartition(partitionOfRecords -> {
    // ConnectionPool is a static, lazily initialized pool of connections
    Connection connection = ConnectionPool.getConnection();
    while (partitionOfRecords.hasNext()) {
      connection.send(partitionOfRecords.next());
    }
    ConnectionPool.returnConnection(connection); // return to the pool for future reuse
  });
});
```

上述例子进一步优化，连接池惰性初始化，并且能够跨批次复用连接对象。

需要注意的地方：

- 输出操作通过懒执行方式操作DStream，正如RDD action通过懒执行的方式操作RDD。具体地看，RDD action和DStream输出操作接收数据的处理。因此，如果你的应用程序没有任何输出操作或者用于输出操作**dstream.foreachRDD()**，但是没有任何**RDD action**操作在**dstream.foreachRDD()**里面，那么什么也不会执行。系统仅仅会接收输入，然后丢弃它们。
- 默认情况下，**DStreams**输出操作是分时执行的，它们按照应用程序的定义顺序按序执行。

### DataFrame和SQL操作

可以很容易地使用**DataFrames**和**SQL**流式地操作数据。需要使用**SparkContext**或者正在使用的**StreamingContext**来创建一个**SparkSession**。这样做的目的就是为了使得驱动程序可以在失败之后进行重启。使用懒加载模式创建单例的**SparkSession**对象。如下，

```java
JavaSparkContext sc = new JavaSparkContext(conf);
JavaStreamingContext ssc = new JavaStreamingContext(sc, Durations.seconds(5));
sc.setLogLevel("WARN");
String sourceService = "192.168.32.61";
int sourcePort1 = 12345;
JavaDStream<String> rs = ssc.socketTextStream(sourceService, sourcePort1)
        .flatMap(l -> Arrays.asList(l.split(" ")).iterator());
rs.foreachRDD((rdd, time) -> {
    SparkSession ss = SparkSession.builder().config(conf).getOrCreate();
    Dataset<Row> df = ss.createDataFrame(rdd.map(w -> new SWord(w)), SWord.class).toDF();

    df.createOrReplaceTempView("word");
    ss.sql("select word.word, count(*) from word group by word").show();
});

ssc.start();
try {
    ssc.awaitTermination();
} catch (InterruptedException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
}

...
package com.embedway.spark.streaming;

public class SWord implements java.io.Serializable{
	private String word;
	public SWord() {
		// TODO Auto-generated constructor stub
	}
	public SWord(String word) {
		// TODO Auto-generated constructor stub
		this.word = word;
	}
	
	public String getWord() {
		return word;
	}

	public void setWord(String word) {
		this.word = word;
	}
	
	
}

```

上述代码统计了单词个数。再不创建实体类的情况下同样可以，使用**StructType**来定义模式。

```java
JavaSparkContext sc = new JavaSparkContext(conf);
JavaStreamingContext ssc = new JavaStreamingContext(sc, Durations.seconds(5));
sc.setLogLevel("WARN");
String sourceService = "192.168.32.61";
int sourcePort1 = 12345;
JavaDStream<String> rs = ssc.socketTextStream(sourceService, sourcePort1)
        .flatMap(l -> Arrays.asList(l.split(" ")).iterator());
rs.foreachRDD((rdd, time) -> {
    SparkSession ss = SparkSession.builder().config(conf).getOrCreate();

    List<StructField> fields = new ArrayList<StructField>();
    fields.add(DataTypes.createStructField("word", DataTypes.StringType, false));
    StructType schema = DataTypes.createStructType(fields);
    Dataset<Row> df = ss.createDataFrame(rdd.map(w -> RowFactory.create(w)), schema).toDF();

    df.createOrReplaceTempView("word");
    ss.sql("select word.word, count(*) from word group by word").show();
});

ssc.start();
try {
    ssc.awaitTermination();
} catch (InterruptedException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
}
```

### 缓存和持久化

**DStream**允许开发人员持久化**streaming**数据在内存中。也就是说，在**DStream**上使用**persist()**方法，自动地把**RDD**持久化至内存中。

**reduceByWindow**、**reduceByKeyAndWindow**、**updateStateByKey**这些都隐式地开启了**persist()**方法，**DStream**生成的窗口操作会自动保存在内存中。

对于通过网络接收数据输入流（如**Kafka**、**Flume**、**Sockets**等），默认的持久性级别为复制两个节点的数据容错。

### Checkpoint

**Spark Streaming**通过checkpoint来容错，以便任务失败之后可以从checkpoint里面恢复。

- 元数据信息checkpoint主要是驱动程序恢复
  - 配置构建streaming应用程序的配置
  - DStream操作streaming程序中的一系列DStream操作
  - 没有完成的批处理，在运行队列中的未完成的批处理

- 数据的checkpoint

  - 保存生成的RDD到一个可靠的存储系统中，常用HDFS，通常有状态的transformation（**updateStateByKey**、**reduceByKeyAndWindow**）（结合多批次的数据），需要做checkpoint。在这样的transformation中，生成的RDD依赖于之前批的RDD，随着时间的推移，这个依赖链的长度会持续增长。在恢复的过程中，为了避免这种无限增长，有状态的transformation的中间RDD将会定时地存储到可靠的存储系统中。以截断依赖链。
如果没有前述状态的transformation简单流应用，可以不开启checkpoint。在这种情况下，从driver故障的恢复是部分恢复（接收到了但是还没有处理的数据将会丢失）。
- 当应用程序第一次启动，新建一个**StreamingContext**，启动所有**Stream**，然后调用**start()**方法。
- 当应用程序因为故障重新启动，它将会从**checkpoint**目录**checkpoint**数据重新创建**StreamingContext**。
如下：

```java
String checkpointDirectory = "file:///home/yanchengyu/streamCheckPoint";
JavaStreamingContext jssc = JavaStreamingContext.getOrCreate(checkpointDirectory, () -> {
    JavaStreamingContext ssc = new JavaStreamingContext(conf, Durations.seconds(5));
    ssc.checkpoint(checkpointDirectory);
    JavaSparkContext sc = ssc.sparkContext();

    sc.setLogLevel("WARN");
    String sourceService = "192.168.32.61";
    int sourcePort = 12345;
    JavaDStream<String> rs = ssc.socketTextStream(sourceService, sourcePort)
            .flatMap(l -> Arrays.asList(l.split(" ")).iterator());
    rs.checkpoint(Durations.seconds(25));
    rs.mapToPair(w -> new Tuple2<>(w, 1))
        .<Integer>updateStateByKey((v, s) -> 
        Optional.of(v.stream().mapToInt(val -> val).sum() + s.or(0)))
        .print();
    return ssc;
});

jssc.start();
try {
    jssc.awaitTermination();
} catch (InterruptedException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
}
```

如果`checkpointDirectory`存在，上下文会将**checkpoint**数据重新创建。如果这个目录不存在，将调用传入的闭包创建一个新的上下文。

除了使用**getOrCreate**，开发者必须保证故障发生时，**driver**处理自动重启。只能通过部署运行应用程序的基础设施来达到该目的。

**RDD**有存储成本，这会导致批数据的处理时间增加。因此，需要小心的设置批处理的时间间隔。建议**checkpoint**时间间隔为批处理时间间隔的5-10倍。

### 累加器和广播变量

**Spark Streaming**无法从**checkpoint**中恢复累加器和广播变量。如果启用**checkpoint**并使用了累加器或广播变量，必须为它们创建Lazy实例化的单实例，以便在驱动程序重新启动失败后重新实例化。

```java
String checkpointDirectory = "file:///home/yanchengyu/baCheckPoint";
JavaStreamingContext jssc = JavaStreamingContext.getOrCreate(checkpointDirectory, () -> {
    JavaStreamingContext ssc = new JavaStreamingContext(conf, Durations.seconds(5));
    ssc.checkpoint(checkpointDirectory);
    JavaSparkContext sc = ssc.sparkContext();
    LongAccumulator acc = SingletonAccul.getInstance(sc);
    sc.setLogLevel("WARN");
    String sourceService = "192.168.32.61";
    int sourcePort = 12345;
    JavaDStream<String> rs = ssc.socketTextStream(sourceService, sourcePort)
            .flatMap(l -> Arrays.asList(l.split(" ")).iterator());
    rs.checkpoint(Durations.seconds(25));
    rs.foreachRDD(rdd -> {
        System.out.println(acc.value());
        acc.add(1);
    });
    return ssc;
});

jssc.start();
try {
    jssc.awaitTermination();
} catch (InterruptedException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
}
```

中断重启抛异常，

```java
19/01/09 14:59:07 ERROR StreamingContext: Error starting the context, marking it as stopped
java.io.IOException: java.lang.UnsupportedOperationException: Accumulator must be registered before send to executor
```

```java
String checkpointDirectory = "file:///home/yanchengyu/batCheckPoint";
JavaStreamingContext jssc = JavaStreamingContext.getOrCreate(checkpointDirectory, () -> {
    JavaStreamingContext ssc = new JavaStreamingContext(conf, Durations.seconds(5));
    ssc.checkpoint(checkpointDirectory);
    JavaSparkContext sc = ssc.sparkContext();

    sc.setLogLevel("WARN");
    String sourceService = "192.168.32.61";
    int sourcePort = 12345;
    JavaDStream<String> rs = ssc.socketTextStream(sourceService, sourcePort)
            .flatMap(l -> Arrays.asList(l.split(" ")).iterator());
    rs.checkpoint(Durations.seconds(5));
    rs.foreachRDD(rdd -> {
        LongAccumulator acc = SingletonAccul.getInstance(rdd.context());
        System.out.println(acc.value());

        rdd.foreach(w -> {System.out.println(w);acc.add(1);});
    });
    return ssc;
});
jssc.start();
try {
    jssc.awaitTermination();
} catch (InterruptedException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
}
```