---
layout: post
title: "Spark RDD 1 --- 基础"
subtitle: "介绍Spark RDD的相关概念和基本操作"
date: 2019-12-06
author: "Cheney.Yin"
header-img: "//imgloc.com/i/iADI63"
tags:
 - Spark
 - RDD
 - 数据分区
 - 闭包
---

# RDD基础

Spark以弹性分布式数据集作为中心，RDD是容错、具备并且操作的元素集合。

创建RDD的两种方式：

1. driver程序中，调用`SparkContext.parallelize()`.
2. 从外部存储引入数据集。

## 并行集合

在spark-shell交互模式下，执行如下命令：

```scala
val distData = sc.parallelize(Array(1,2,4,5))
distData.partitions.foreach(x => println(x))
```

结果如下：

```shell
org.apache.spark.rdd.ParallelCollectionPartition@691
org.apache.spark.rdd.ParallelCollectionPartition@692
org.apache.spark.rdd.ParallelCollectionPartition@693
org.apache.spark.rdd.ParallelCollectionPartition@694
```

`distData`拥有4个分区。

执行如下命令求和：

```scala
distData.reduce((s, x) => s + x)
res7: Int = 12
```

并行集合有一个很重要的参数是partitions分区数量，它可以用来切割dataset(数据集)。Spark将在集群中的每个分区上运行一个任务。一般情况下，Spark会尝试根据机器情况来自动设置分区数量。当然，我们也可以指定参数来设置分区数量，如下：

```scala
val distData1 = sc.parallelize(Array(1,2,3,4,5,6,7,8))

val distData2 = sc.parallelize(Array(1,2,3,4,5,6,7,8), 2)
```

结果如下：

``` 
scala> distData1.partitions.foreach(x => println(x))
org.apache.spark.rdd.ParallelCollectionPartition@6e3
org.apache.spark.rdd.ParallelCollectionPartition@6e4
org.apache.spark.rdd.ParallelCollectionPartition@6e5
org.apache.spark.rdd.ParallelCollectionPartition@6e6

scala> distData2.partitions.foreach(x => println(x))
org.apache.spark.rdd.ParallelCollectionPartition@70c
org.apache.spark.rdd.ParallelCollectionPartition@70d
```

## 外部数据集

Spark可以同Hadoop支持的任何存储源中建立分布式数据集，包括本地文件系统，HDFS，Cassandra，HBase，Amazon S3等等。

Spark支持文本文件、SequenceFiles以及任何其它的Hadoop InputFormat。

可以使用SparkContext的textFile方法创建文本文件的RDD。此方法需要一个文件的URI(计算机上的本地路径，hdfs://，s3n://等等URI)，并且读取它们作为一个lines的集合。调试如下：

```scala
scala> val distFile = sc.textFile("spark-shell.cmd")
distFile: org.apache.spark.rdd.RDD[String] = spark-shell.cmd MapPartitionsRDD[1] at textFile at <console>:24

scala> distFile.partitions.foreach(x => println(x))
org.apache.spark.rdd.HadoopPartition@3c1
org.apache.spark.rdd.HadoopPartition@3c2

scala> val distFile = sc.textFile("spark-shell.cmd", 4)
distFile: org.apache.spark.rdd.RDD[String] = spark-shell.cmd MapPartitionsRDD[3] at textFile at <console>:24

scala> distFile.partitions.foreach(x => println(x))
org.apache.spark.rdd.HadoopPartition@3ff
org.apache.spark.rdd.HadoopPartition@400
org.apache.spark.rdd.HadoopPartition@401
org.apache.spark.rdd.HadoopPartition@402

```

在创建后，distFile可以使用dataset的操作。通过如下命令统计行数，

```scala
scala> distFile.map(l => 1).reduce((s,n) => s + n)
res3: Int = 23
```

使用Spark读取文件的注意事项：

- 如果使用本地文件系统路径，所有工作节点的相同路径下该文件必须能够访问。复制文件到所有工作节点上，或者使用共享的网络挂在文件系统。

- 所有Spark中基于文件的输入方法，包括textFile，支持目录、压缩文件或者通配符来操作。例如，

  ```shell
  textFile("/my/directory")
  
  textFile("/my/directory/*.txt")
  
  textFile("/my/directory/*.gz")
  ```

- textFile方法也可以通过第二个可选的参数来控制该文件的分区数量。默认情况下，Spark为文件的每个block创建一个分区（HDFS中块大小默认64M），当然你也可以通过传递一个较大的值来要求一个较高的分区数量。PS.分区数量不能小于块的数量。

除了文本文件外，Spark的Scala API也支持一些其它的数据格式：
- SparkContext.wholeTextFiles可以读取包含多个小文件的目录，并返回它们的每一个(filename, content)对。这与textFile形成对比，它的每一文件中的每一行返回一个记录。
- 针对SequenceFiles，使用SparkContext的SequenceFile[K, V]方法，其中K、V指的是它们在文件中的类型。这些应该是Hadoop中Writeable接口的子类，例如IntWritable和Text。此外，Spark可以让您为一些常见的Writables指定原生类型。例如，SequenceFile[Int, String]会自动读取IntWritables和Texts。
- 针对其它的Hadoop InputFormats，您可以使用SparkContext。hadoopRDD方法，它接受一个任意JobConf和Input format类、key类和value类。通过相同的方法你可以设置你Hadoop Job的输入源。你还可以使用基于"new"和MapReduce API(org.apache.hadoop.mapreduce)来使用SparkContext.newAPIHadoopRDD以设置InputFormats。
- RDD.saveAsObjectFile和SparkContext.objectFile支持使用简单的序列化的Java Object来保存RDD。虽然这不像Avro这种专用的格式一样高效，但是它提供了一种更简单的方式来保存任何RDD。

### HadoopRDD分区方法

|      | #1   | #2   | #3   | #4   |
| ---- | ---- | ---- | ---- | ---- |
| #1   | a    | b    | c    | d    |
| #2   | \n   | 3    | g    | f    |
| #3   | v    | e    | t    | d    |
| #4   | d    | \n   | d    | 4    |
| #5   | \n   | o    | z    | y    |
| #6   | y    | y    | c    | \n   |

上表现某个文件的存储内容，假设文件内容按照‘\n’划分记录，并且最小分区数为2，那么HadoopRDD的分区方法如下：

HadoopRDD首先会根据文件总长度/最小分区数来预分区，上述文件预分区结果如下：

partition(0) := [#1, #1] -> (#4, #1)

partition(1) := [#4, #1] -> (#7, #1)

然后进行实际分区，实际分区会在预分区的基础上作出调整，并且各个分区的计算是独立的。

首先，确定每个分区的起点。首个分区的起点，如partition(0)，不做调整。非首个分区的起点，需要重新计算，计算方式是从与分区起点开始(不包含该点)，向后搜索，直至找到首个记录分割符或到达文件末尾。若找到分割符，则起点为分割符的后继位置。若计算的起点位置超出文件范围，后者未找到分割符，则分区为空。

然后，确定每个分区的终点，需要从预分区终点(不包含该点)，向后搜索，直至找到首个记录分隔符或到达文件末尾。若找到分割符或到达文件末尾，则终点为搜索停止处的后继位置。

以上述文件为例，

计算起点

partition(0) := [#1, #1] -> (#4, #1)    =>    [#1, #1] -> (#4, #1) 

partition(1) := [#4, #1] -> (#7, #1)    =>    [#4, #3] -> (#7, #1)

计算终点

partition(0) := [#1, #1] -> (#4, #1)    =>    [#1, #1] -> (#4, #3) 

partition(1) := [#4, #3] -> (#7, #1)    =>    [#4, #3] -> (#7, #1)

若最小分区数为3，则预分区为：

partition(0) := [#1, #1] -> (#3, #1)

partition(1) := [#3, #1] -> (#5, #1)

partition(2) := [#5, #1] -> (#7, #1)

计算起点

partition(0) := [#1, #1] -> (#3, #1)	=>	[#1, #1] -> (#3, #1)

partition(1) := [#3, #1] -> (#5, #1)	=>	[#4, #3] -> (#5, #1)

partition(2) := [#5, #1] -> (#7, #1)	=>	[#7, #1] -> (#5, #1)

计算终点

partition(0) := [#1, #1] -> (#3, #1)	=>	[#1, #1] -> (#4, #3)

partition(1) := [#4, #3] -> (#5, #1)	=>	[#4, #3] -> (#7, #1)

partition(2) := [#7, #1] -> (#5, #1)	=>	[#7, #1] -> (#7, #1)	：为空

若最小分区数为6，则预分区为：

partition(0) := [#1, #1] -> (#2, #1)

partition(1) := [#2, #1] -> (#3, #1)

partition(2) := [#3, #1] -> (#4, #1)

partition(3) := [#4, #1] -> (#5, #1)

partition(4) := [#5, #1] -> (#6, #1)

partition(5) := [#6, #1] -> (#7, #1)

计算

partition(0) := [#1, #1] -> (#2, #1)	=>	[#1, #1] -> (#4, #3)

partition(1) := [#2, #1] -> (#3, #1)	=>	[#4, #3] -> (#7, #1)

partition(2) := [#3, #1] -> (#4, #1)	=>	[#4, #3] -> (#7, #1)

partition(3) := [#4, #1] -> (#5, #1)	=>	[#4, #3] -> (#7, #1)

partition(4) := [#5, #1] -> (#6, #1)	=>	[#7, #1] ->  (#7 #1)

partition(5) := [#6, #1] -> (#7, #1)	=>	[#7, #1] -> (#7, #1)

partition(1)、partition(2)、partition(3)等价，partition(4)、partition(5)为空。

## RDD操作

如下命令使用RDD的基本操作实现了文件长度统计，如下：

```scala
scala> val distFile = sc.textFile("spark-shell.cmd")
distFile: org.apache.spark.rdd.RDD[String] = spark-shell.cmd MapPartitionsRDD[11] at textFile at <console>:24

scala> distFile.map(l => l.length).reduce((s, n) => s + n)
res7: Int = 987
```

首先定义了一个基本的RDD，但是这个数据集没有立即加载到内存。

`distFile.map(l => l.length)`是map transformation操作。由于laziness，操作不会立即执行。

最后的`reduce`操作，是一个action。此时，Spark分发计算任务到不同的机器上运行，每个机器都运行在map的一部分，并在本地运行reduce，仅返回聚合结果后给驱动程序。

若希望后续再次使用`distFile`，可以执行如下命令：

```scala
scala> distFile.persist()
res8: distFile.type = spark-shell.cmd MapPartitionsRDD[11] at textFile at <console>:24
```

这样在其它同`reduce`类似的action操作执行前，`distFile`不需要重新计算。

## 向Spark传递函数

当驱动程序在集群上运行时，Spark的API在很大程度上依赖于传递函数。有俩各种推荐方式来做到这一点：

- 匿名函数的语法Anonymous Function syntax， 它可以用于短的代码片断。

- 在全局单例对象中的静态方法。例如，你可以定义对象MyFunctions然后传递MyFunctions.func1，具体如下：

  ```scala
  scala> object MyFunctions {
       | def func1(s: String): String = {
       | return s + "_hello_";
       | }
       | }
  defined object MyFunctions
  
  scala> val distFile = sc.textFile("spark-shell.cmd")
  distFile: org.apache.spark.rdd.RDD[String] = spark-shell.cmd MapPartitionsRDD[1] at textFile at <console>:24
  
  scala> distFile.map(MyFunctions.func1).collect()
  res0: Array[String] = Array(@echo off_hello_, _hello_, rem_hello_, rem Licensed to the Apache Software Foundation (ASF) under one or more_hello_, rem contributor license agreements.  See the NOTICE file distributed with_hello_, rem this work for additional information regarding copyright ownership._hello_, rem The ASF licenses this file to You under the Apache License, Version 2.0_hello_, rem (the "License"); you may not use this file except in compliance with_hello_, rem the License.  You may obtain a copy of the License at_hello_, rem_hello_, rem    http://www.apache.org/licenses/LICENSE-2.0_hello_, rem_hello_, rem Unless required by applicable law or agreed to in writing, software_hello_, rem distributed under the License is distributed on an "AS IS" BASIS,_hello_, rem WITHOUT WARRAN...
  ```

  注意，虽然也可以传递一个类的实例的方法的引用，这需要发送整个对象，包括类的其它方法。例如，考虑：

```scala
scala> class MyClass extends Serializable { 
     |   import org.apache.spark.rdd.RDD
     |   def func1(s: String): String = { return s + "_hello_";}
     |   def doStuff(rdd:RDD[String]): RDD[String] = {rdd.map(func1)}
     |   }
defined class MyClass

scala> val myInstance01 = new MyClass()
myInstance01: MyClass = MyClass@259ae1a9

scala> myInstance01.doStuff(distFile).collect()
res4: Array[String] = Array(@echo off_hello_, _hello_, rem_hello_, rem Licensed to the Apache Software Foundation (ASF) under one or more_hello_, rem contributor license agreements.  See the NOTICE file distributed with_hello_, rem this work for additional information regarding copyright ownership._hello_, rem The ASF licenses this file to You under the Apache License, Version 2.0_hello_, rem (the "License"); you may not use this file except in compliance with_hello_, rem the License.  You may obtain a copy of the License at_hello_, rem_hello_, rem    http://www.apache.org/licenses/LICENSE-2.0_hello_, rem_hello_, rem Unless required by applicable law or agreed to in writing, software_hello_, rem distributed under the License is distributed on an "AS IS" BASIS,_hello_, rem WITHOUT WARRAN...
```

注意，这里`class MyClass extends Serializable`，没有继承将导致对象无法在集群内发送传输，驱动程序抛出如下异常：

```scala
org.apache.spark.SparkException: Task not serializable
  at org.apache.spark.util.ClosureCleaner$.ensureSerializable(ClosureCleaner.scala:340)
  at org.apache.spark.util.ClosureCleaner$.org$apache$spark$util$ClosureCleaner$$clean(ClosureCleaner.scala:330)
  at org.apache.spark.util.ClosureCleaner$.clean(ClosureCleaner.scala:156)
  at org.apache.spark.SparkContext.clean(SparkContext.scala:2294)
  at org.apache.spark.rdd.RDD$$anonfun$map$1.apply(RDD.scala:370)
  at org.apache.spark.rdd.RDD$$anonfun$map$1.apply(RDD.scala:369)
  at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:151)
  at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:112)
  at org.apache.spark.rdd.RDD.withScope(RDD.scala:362)
  at org.apache.spark.rdd.RDD.map(RDD.scala:369)
  at MyClass.doStuff(<console>:14)
  ... 48 elided
Caused by: java.io.NotSerializableException: MyClass
Serialization stack:
	- object not serializable (class: MyClass, value: MyClass@73476e2d)
	- field (class: MyClass$$anonfun$doStuff$1, name: $outer, type: class MyClass)
	- object (class MyClass$$anonfun$doStuff$1, <function1>)
  at org.apache.spark.serializer.SerializationDebugger$.improveException(SerializationDebugger.scala:40)
  at org.apache.spark.serializer.JavaSerializationStream.writeObject(JavaSerializer.scala:46)
  at org.apache.spark.serializer.JavaSerializerInstance.serialize(JavaSerializer.scala:100)
  at org.apache.spark.util.ClosureCleaner$.ensureSerializable(ClosureCleaner.scala:337)
  ... 58 more
```

类似地，

```scala
scala> class MyClass extends Serializable{
     |   import org.apache.spark.rdd.RDD
     |   val field = "Hello"
     |   def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(x => field + x) }
     | }
defined class MyClass

scala> val myInstance01 = new MyClass()
myInstance01: MyClass = MyClass@2c78771b

scala> myInstance01.doStuff(distFile).collect()
res6: Array[String] = Array(Hello@echo off, Hello, Hellorem, Hellorem Licensed to the Apache Software Foundation (ASF) under one or more, Hellorem contributor license agreements.  See the NOTICE file distributed with, Hellorem this work for additional information regarding copyright ownership., Hellorem The ASF licenses this file to You under the Apache License, Version 2.0, Hellorem (the "License"); you may not use this file except in compliance with, Hellorem the License.  You may obtain a copy of the License at, Hellorem, Hellorem    http://www.apache.org/licenses/LICENSE-2.0, Hellorem, Hellorem Unless required by applicable law or agreed to in writing, software, Hellorem distributed under the License is distributed on an "AS IS" BASIS,, Hellorem WITHOUT WARRANTIES OR CONDITIONS OF A...
```

同样需要继承`Serializable`，这是由于`def doStuff(rdd:RDD[String]): RDD[String] = {rdd.map(func1)}`和`def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(x => field + x) }`，在`map`算子中都需要访问对象。如下方式则不必继承`Serializable`，

```scala
scala> class MyClass {
     |   import org.apache.spark.rdd.RDD
     |   val field = "Hello"
     |   def doStuff(rdd: RDD[String]): RDD[String] = { 
     |   val field_ = this.field
     |   rdd.map(x => field_ + x) 
     |   }
     | }
defined class MyClass

scala> val myInstance01 = new MyClass()
myInstance01: MyClass = MyClass@3aede2ff

scala> myInstance01.doStuff(distFile).collect()
res7: Array[String] = Array(Hello@echo off, Hello, Hellorem, Hellorem Licensed to the Apache Software Foundation (ASF) under one or more, Hellorem contributor license agreements.  See the NOTICE file distributed with, Hellorem this work for additional information regarding copyright ownership., Hellorem The ASF licenses this file to You under the Apache License, Version 2.0, Hellorem (the "License"); you may not use this file except in compliance with, Hellorem the License.  You may obtain a copy of the License at, Hellorem, Hellorem    http://www.apache.org/licenses/LICENSE-2.0, Hellorem, Hellorem Unless required by applicable law or agreed to in writing, software, Hellorem distributed under the License is distributed on an "AS IS" BASIS,, Hellorem WITHOUT WARRANTIES OR CONDITIONS OF A...
```

**在驱动程序中，预先拷贝了`MyClass.field`，`map`算子不再访问`MyClass`的实例。**

注意：Scala的`Serializable`实质上继承自`java.io.Serializable`。

## 理解闭包

在集群中执行代码时，一个关于Spark更难的事情是理解变量和方法的范围和生命周期。如下示例是一种常见的混淆变量和方法范围的案例。

```scala
scala> var counter = 0
counter: Int = 0
scala> var distFile = sc.textFile("spark-shell.cmd")
distFile: org.apache.spark.rdd.RDD[String] = spark-shell.cmd MapPartitionsRDD[6] at textFile at <console>:25
scala> distFile.foreach(line => counter + 1)
scala> println("Counter = " + counter)
Counter = 0
```

上述代码行为是不确定的，并且可能无法按预期正常工作。Spark执行作业时，会分解RDD操作到每个执行者。在执行之前，Spark计算task的closure。而闭包包含了RDD上的执行者必须能够访问的变量和方法。闭包被序列化发送到每个执行器。

闭包的变量副本发给了每个执行器，当counter被foreach函数引用时，它已经不再是驱动节点的counter。虽然在驱动节点仍然有一个counter在内存中，但是对执行器是不可见的，执行器访问到的只是序列化的闭包提供的副本，foreach中被累加的counter也只是执行器局部访问的副本。所以counter仍为0。

在本地模式，某些情况下的foreach功能实际上是在同一JVM上的驱动程序执行的，并引用了同一原始的计数器，实际上可能更新。

为了确保在这些场景下达到目的，可以使用Accumulator累加器。当一个执行的任务分配到集群中的各个worker节点时，Spark的累加器提供了安全的更新变量机制。

一般情况下，闭包结构或本地定义的方法，不应该用于修改全局变量。

如`rdd.foreach(println)`或`rdd.map(println)`是不能保证产生预期输出的，如在集群模式下，`println`使用的`stdout`是执行器本地的`stdout`，在驱动程序上是看不到效果的。打印输出可以使用如下两种方法，

```scala
distFile.collect().foreach(x => println(x))
distFile.take(10).foreach(x => println(x))
```

 其中，`take`方法限制了聚合数据的元素的数量。

## 使用Key-Value对

虽然大多数Spark操作工作在包含任何类型对象的RDDs上，只有少数特殊的操作可以用于Key-Value对的RDDs。最常见的是分布式“shuffle”操作，如通过元素的key来进行`grouping`或`aggregating`操作。

在Scala中，这些操作时可用Tuple2对象。在`PairRDDFunctions`类中该Key-Value对操作有效，其中围绕元组的RDD自动包装。

例如，下面代码使用的Key-Value对的 `reduceByKey`操作统计文本文件的单词数量。

```scala
scala> val distFile = sc.textFile("spark-shell.cmd")
distFile: org.apache.spark.rdd.RDD[String] = spark-shell.cmd MapPartitionsRDD[10] at textFile at <console>:25

scala> val pairs = distFile.flatMap(l => l.split(" ")).map(w => (w, 1))
pairs: org.apache.spark.rdd.RDD[(String, Int)] = MapPartitionsRDD[13] at map at <console>:27

scala> val counts = pairs.reduceByKey((s, n) => s + n)
counts: org.apache.spark.rdd.RDD[(String, Int)] = ShuffledRDD[14] at reduceByKey at <console>:29

scala> counts.take(10).foreach((w) => println(w))
(entry,1)
(Unless,1)
(this,3)
(licenses,1)
(under,4)
(KIND,,1)
(is,2)
(polluting,1)
(rem,18)
(CONDITIONS,1)

```

也可以使用`counts.sortByKey()`来按照字母表顺序排序。

```scala
scala> counts.sortByKey().take(10).foreach((w) => println(w))
(,8)
("%~dp0spark-shell2.cmd",1)
("AS,1)
("License");,1)
(%*,1)
((ASF),1)
((the,1)
(/C,1)
(/E,1)
(/V,1)
```

注意：使用自定义对象作为Key-Value对操作的key时，必须确保自定义`equal()`方法有一个`hashCode()`方法相匹配。