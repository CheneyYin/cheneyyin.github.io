---
layout: post
title: "Spark共享变量"
subtitle: "介绍Spark共享变量的相关概念和使用方法"
date: 2020-01-05
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags:
 - Spark
 - 共享变量
 - 广播变量
 - 累加器
---

# 共享变量

通常情况下，一个传递给**Spark**操作（例如，`map`和`reduce`）的函数`func`是在远程的集群节点上执行的。该函数`func`在多个节点执行过程中使用的变量是同一个变量的多个副本。这些变量以副本的方式拷贝到每台机器上，并且各个远程机器上变量的更新并不会传播回**driver program**。通用且支持**read-write**的共享变量在任务间是不能胜任的。所以，**Spark**提供了两种特定类型的共享变量：**broadcast variables**（广播变量）和**accumulators**（累加器）。

## Broadcast Variables（广播变量）

**Broadcast Variables**（广播变量）允许程序员将一个**read-only**变量缓存到每台机器上，而不是给任务传递一个副本。广播变量可以使用一种高效的方式给每个节点传递一份比较大的**input dataset**副本。在使用广播变量时，**Spark**也尝试使用高效广播算法分发广播变量，来降低通信成本。
**Spark**的**action**操作是通过一系列的**stage**进行执行的，这些**stage**是通过分布式的“**shuffle**”操作进行拆分的。Spark会自动广播出每个**stage**内任务所需的公共数据。这种情况下广播的数据使用序列化的形式缓存，并在每个任务运行前进行反序列。这也意味着，只有在跨越多个**stage**的多个任务会使用相同的数据，或者在使用反序列化形式的数据特别重要的情况下，使用广播变量会有比较好的效果。
广播变量通过在一个变量**v**上调用`SparkContext.broadcast(v)`方法来进行创建。广播变量**v**的一个**wrapper**（包装器），可以通过调用**value**方法来访问它的值。示例如下，
```scala
scala> val broadCastVal = sc.broadcast(Array(1, 2, 3, 4))
broadCastVal: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

scala> broadCastVal.value
res0: Array[Int] = Array(1, 2, 3, 4)
```
在创建广播变量之后，在集群上执行的所有的函数中，应该使用该广播变量代替原来的**v**值，所以节点上的**v**最多分发一次。另外，对象**v**在广播后不应该再被修改，以保证分发到所有的节点上的广播变量具有相同的值。
如下示例为文本文件的每一行增加了`spark.serialization`，`spark.context`。
```scala
val bcVals = Array("spark.serialization", "spark.context")
val broadCastVals = sc.broadcast(bcVals)
val distFile = sc.textFile("spark-shell.cmd")
distFile.map(l => { broadCastVals.value.fold(l){(c, s) => c + s}})
```
- 通过对一个类型**T**的对象调用`SparkContext.broadcast`创建出一个`Broadcast[T]`对象。任何可序列化的类型都可以这么实现。

- 通过`value`属性访问该对象的值。

- **变量只会被发到各个节点一次，应作为只读值处理**。（Spark把闭包中引用的变量发送到工作节点上，虽然很方便，但是也很低效。原因如下（1）：默认的任务发射机制是专门为小任务进行优化的；（2）：事实上，你可能会在多个并行操作中使用同一个变量，但是Spark会为每个操作分别发送。）

## 累加器

Accumulators（累加器）是一个仅可以执行"**added**"（添加）的变量来通过一个关联和交换操作，因此可以高效地执行支持并行。累加器可以用于实现**counter**（计数）或者**sums**（求和）。原生**Spark**支持数值型的累加器，并且程序员可以添加新的支持类型。
创建**accumulators**并命名后，在**Spark**的UI界面上将会显示它。这样可以帮助理解正在运行阶段的运行情况。
可以通过调用`SparkContext.longAccumulator()`或`SparkContext.doubleAccumulator()`方法创建数值类型的**accumulator**以分别累加**Long**或**Double**类型的值。集群上正在运行的任务就可以使用**add**方法来累计数值。然而，它们不能够读取它的值。只有**driver program**才可以使用**value**方法读取累加器的值。
下面的代码展示了一个**accumulator**被用于一个数字中的元素求和。
```scala
val distArray = sc.parallelize(Array(1, 2, 3, 4))
val accum = sc.longAccumulator("My Accumulator")
distArray.foreach(v => accum.add(v))
accum.value
res8: Long = 10
```
上面的代码示例使用的是**Spark**内置的**Long**类型的累加器，程序员可以通过继承`AccumulatorV2`类创建新的累加器类型。**AccumulatorV2**抽象类有几个需要**override**的方法：**reset**方法可将累加器重置为**0**，**add**方法可将其它值添加到累加器中，**merge**方法可将其它同类型的累加器合并为一个。其它需要重写的方法参考**scala API**文档。
累加器的更新只发生在**action**操作中，**Spark**保证每个任务只更新一次累加器，例如，重启任务不会更新值。在**transformation**中，用户需要注意的是，如果**task**或者**job stages**重新执行，每个任务的更新操作可能会执行多次。
累加器不会改变**Spark lazy evaluation**（懒加载）的模式。如果累加器在**RDD**中的一个操作中进行更新，它们的值仅被更新一次，**RDD**被作为**action**的一部分来计算。因此，在一个像`map()`这样的**transformation**时，累加器的更新并没有执行。
```scala
val accum = sc.longAccumulator("counter")
val distFile = sc.textFile("spark-shell.cmd")
val mapRes = distFile.map(l => {accum.add(1);l;})
scala> accum.value
res19: Long = 0
mapRes.collect()
scala> accum.value
res21: Long = 23
```

- 对工作节点而言，累加器是**写入变量**，不可读。
- 在**action**操作中使用的累加器，**Spark**只会把每个任务对各累加器的修改应用**一次**。转换操作中使用累加器，在任务重启后累加器更新操作会再次重复执行。建议**转换操作中使用累加器只用于调试**。