---
layout: post
title: "Spark RDD 2 --- 操作"
subtitle: "介绍RDD的Transformation、Action和Shuffle操作。"
date: 2019-12-12
author: "Cheney.Yin"
header-img: "//imgloc.com/i/iADI63"
tags:
 - Spark
 - RDD
 - Transformation
 - Action
 - Shuffle
---

# RDD操作

Spark RDD常见操作分为三类： Transformation、Action和Shuffle。

## Transformations 转换操作

| Transformation(转换)                                 | 含义                                                         | 示例                                                         |
| :--------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| map(func)                                            | 返回一个新的distributed dataset，它由每个source中的元素应用一个函数func来生成。 |                                                              |
| filter(func)                                         | 返回一个新的distributed dataset，对source中每个元素应用一个函数func来筛选，结果由返回true的元素组成。 |                                                              |
| flatMap(func)                                        | 和map类似，但是每个输入的item可以被映射成0个或多个输出的items。(每个item会返回一个Seq) |                                                              |
| mapPartitions(func)                                  | 和map类似，但是单独运行在每个RDD的分区上，所以在一个类型为T的RDD上运行时，func必须是`Iterator<T> => Iterator<U>`类型。 | 统计文件行数行数：`distFile.mapPartitions(iter => iter.map(l => 1)).reduce((s, n) => s + n)` |
| mapPartitionsWithIndex(func)                         | 和mapPartitions类似，但是需要提供一个代表分区的index的整型值作为参数的func，所以在T类型的RDD上运行时func必须是`(Int, Iterator<T>) => Iterator<U>`类型。 | 统计每个分区的行数(非最优)：`distFile.mapPartitionsWithIndex((idx, iter) => iter.map(l => (idx, 1))).reduceByKey((s, n) => s + n)` |
| sample(withReplacement, fraction, seed)              | 采样，withReplacement(是否有放回)、fraction(采样百分比)、seed(使用指定的随机数生成器的种子) | `distFile.sample(false, 0.50)`                               |
| union(otherDataset)                                  | 返回一个新的数据集，它包含source数据集合other数据集。        | `distFile2.union(distFile)`                                  |
| intersection(otherDataset)                           | 返回一个新的数据集，它包含source数据集和other数据集的交集    | `distFile2.intersection(distFile)`                           |
| distinct([numTasks])                                 | 返回一个新的dataset它含了source数据集去重的元素。(使用对象的hashcode来判断对象相等，distinct是shuffle操作)，numTasks指定结果分区数量。 | `distFile.distinct(1)`                                       |
| groupByKey([numTasks])                               | 在一个`(K, V)`对的数据集上调用时，返回一个`(K, Iterable<V>)`的数据集。注意：如果分组是为了在每个key上执行聚合操作（例如，`sum`和`average`），此时使用`reduceByKey`或`aggregateByKey`来计算，性能会更好。注意：默认情况下的并行度取决于父RDD的分区数。可以传递一个可选的`numTasks`参数来设置不同的任务数。 | `distFile.flatMap(l => l.split(" ")).map(w => (w, 1)).groupByKey()` |
| reduceByKey(func, [numTasks])                        | 在一个`(K, V)`对的数据集上调用，返回一个`(K, V)`对的数据集，它的值会针对每个key使用指定的`reduce`函数`func`来聚合，它必须为`(V, V) => V`类型。像groupByKey一样，可以配置`reduce`任务数量。 | `distFile.flatMap(l => l.split(" ")).map(w => (w, 1)).reduceByKey((s, n) => s + n, 3)` |
| aggregateByKey(zeroValue)(seqOp, combOp, [numTasks]) | 在一个`(K, V)`对的数据集上调用，返回一个`(K, U)`对的数据集，它的值会针对每个key使用指定的`combine`函数和一个中间的`zeroValue`来聚合，它必须为`(V, V) => V`类型。为了避免不必要的配置，可以使用一个不同与input value类型的aggregated value类型。`seqOp`分区内操作，为`(U, V) => U`类型。`combOp`为分区间操作，为`(U, U) => U`类型。返回`(K, U)`类型的`RDD`。 | `distFile.flatMap(l => l.split(" ")).map(w => (w, 1)).aggregateByKey(("",0))((a, e) => (a._1 + "-" + e, a._2 + 1), (a, e) => (a._1 + "` |
| sortByKey([ascending], [numTasks])                   | 在一个`(K, V)`对的数据集上调用，其中K实现了Ordered，返回一个按照keys升序或降序的`(K, V)`对的数据集。 | `distFile.flatMap(l => l.split(" ")).map(w => (w, 1)).reduceByKey((s, n) => s + n).sortByKey()` |
| join(otherDataset, [numTasks])                       | 在`(K,V)`和`(K,W)`类型的数据集上调用时，返回一个`(K,(V,W))`对的数据集，它拥有每个key中所有的元素对。Outer joins可以通过leftOuterJoin，RightOuterJoin和fullOuterJoin来实现。 | `val data1 = sc.textFile("spark-shell.cmd").flatMap(l => l.split(" ")).map(w => (w, 1)).reduceByKey((s, n) => s + n)`;`val data2 = sc.textFile("pyspark.cmd").flatMap(l => l.split(" ")).map(w => (w, 1)).reduceByKey((s, n) => s + n)`;`data1.join(data2)` |
| cogroup(otherDataset, [numTasks])                    | 在`(K,V)`和otherDataset上调用，返回一个`(K,(Iterable<V>, Iterable<W>))`元组的数据集。这个操作也调用了`groupWith`。类似`Full Outer Join`。 |                                                              |
| cartesian(otherDataset)                              | 在一个`T`和`U`类型的数据集上调用，返回一个`(T, U)`对类型的数据集。（所有元素对，即**笛卡儿积**）。注意：结果的分区数为两个数据集分区数的积。 |                                                              |
| pipe(command, [envVars])                             | 通过使用shell命令来将每个RDD的分区给pipe。例如，perl或bash脚本。RDD的元素会被写入进程的标准输入(stdin)，并且lines输出到它的标准输出(stdout)被作为一个字符串型RDD的String返回。 |                                                              |
| coalesce(numPartitions)                              | 降低RDD的分区数量。                                          |                                                              |
| repartition(numPartitions)                           | 重洗牌，RDD中的数据以创建或者更多的分区，并将每个分区中的数据尽量保持均匀。该操作总是通过网络来`shuffles`所有数据。 |                                                              |
| repartitionAndSortWithinPartitions(partitioner)      | 根据给定的partitioner(分区器)对RDD进行重新分区，并在每个结果分区中，按照key值对记录排序。这比每个分区中先调用repartition然后再排序效率更高，因为它可以将排序过程推送到shuffle操作的机器上进行。 | `distFile.flatMap(l => l.split(" ")).map(w => (w, 1)).repartitionAndSortWithinPartitions(new HashPartitioner(3))` |

关于`coalesce`、`repartition`的区分：

```scala
def
coalesce(numPartitions: Int, shuffle: Boolean = false, partitionCoalescer: Option[PartitionCoalescer] = Option.empty)(implicit ord: Ordering[T] = null): RDD[T]

def repartition(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T] = withScope {
    coalesce(numPartitions, shuffle = true)
}
```

- repartition是coalesce执行shuffle的特殊版本。
- coalesce通过在计算节点上合并分区，便可达到减少分区的目的。
- 若父RDD和子RDD分区数量相差较大，使用shuffle来并行处理更高效。

## Actions动作

| Action                                   | 说明                                                         | 例子                                                         |
| ---------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| reduce(func)                             | 使用函数func聚合数据集中的元素。func输入两个元素，返回一个元素。这个函数应该是可以交换和关联的，这样才能保证它可以被并行地正确计算。 `reduce(f: (T, T) => T): T` | `distFile.reduce((d, l) => {d + "`                           |
| collect()                                | 在驱动程序中，以一个数组的形式返回数据集的所有元素。这在返回足**够小**的数据子集的过滤器或其它操作之后通常是有用的。 | `distFile.filter(l => l.length < 10).collect()`              |
| count                                    | 返回数据集中元素个数                                         | `distFile.count()`                                           |
| first()                                  | 返回数据集中的第一个元素。类似`take(1)`。                    | `distFile.first()`                                           |
| take(n)                                  | 将数据集中的前n个元素作为一个数组返回。                      | `distFile.take(2)`                                           |
| takeSample(withReplacement, num, [seed]) | 为一个数据集随机抽样，返回一个包含num个随机抽样元素的数组参数withReplacement指定是否有放回抽样，参数seed指定生成随机数的种子。 | `distFile.flatMap(l => l.split(" ")).takeSample(false, 4)`   |
| takeOrdered(n, [ordering])               | 返回RDD按照自然顺序或自定义比较器排序后的前n个元素。         | `distFile.takeOrdered(4)(Ordering[Int].reverse.on(l => l.length))` |
| saveAsTextFile(path)                     | 将数据集中元素以文本文件(或文本文件集合)的形式写入本地文件系统、HDFS或其它Hadoop支持的文件系统中的给定目录中。Spark将对每个元素调用toString方法，将数据元素转换为文本文件中的一行记录。 | `distFile.flatMap(l => l.split(" ")).map(w => (w, 1)).reduceByKey((s, n) => {s + n}).saveAsTextFile("/home/yanchengyu/counts")` |
| saveAsSequenceFile(path)                 | 将数据集中的元素以Hadoop SequenceFile的形式写入本地文件系统、HDFS或其它Hadoop支持的文件系统指定的路径中。该操作可以在实现了Hadoop的Writable接口键值对的RDD上使用。在Scala中，它可以隐式转化为Writable的类型（Spark包括了基本类型的转换，例如Int、Double、String） | `distFile.flatMap(l => l.split(" ")).map(w => (w, 1)).reduceByKey((s, n) => s + n).saveAsSequenceFile("hdfs://localhost:9000/count")` |
| saveAsObjectFile(path)                   | 使用Java序列化以简单的格式编写数据集的元素，然后使用`SparkContext.objectFile()`进行加载。 | `distFile.flatMap(l => l.split(" ")).map(w => (w, 1)).reduceByKey((s, n) => s + n).saveAsObjectFile("/home/yanchengyu/count.object");sc.objectFile[(String, Int)]("/home/yanchengyu/count.object");` |
| countByKey()                             | 仅适用于(K, V)类型的RDD。返回具有每个Key的计数的(K, Int)对的Hashmap。 | `distFile.flatMap(l => l.split(" ")).map(w => (w, 1)).countByKey()` |
| foreach(func)                            | 对数据集中每个元素运行函数func。这通常用于副作用（side effects），例如更新一个累加器（Accumulator）或者与外部存储系统进行交互。注意：修改除foreach()外的累加器以外的变量可能会导致未定义的行为。func: (T) => Unit |                                                              |

## Shuffle操作

Spark某些操作会触发shuffle。shuffle是spark重新分配数据的一种机制，使得这些数据可以跨不同的区域进行分组。这通常涉及在执行器和机器之间拷贝数据，这使得shuffle成为一个复杂的、代价高的操作。

### 背景

为了明白shuffle操作的过程，我们以`reduceByKey`为例。`reduceByKey`操作产生一个新的RDD，其中key相同的所有值组合成为一个tuple（key以及与key相关联的所有值在`reduce`函数上的执行结果。）但问题是，一个key的所有值不一定都在同一个分区里，甚至不在同一台机器上，但是它们必须被共同计算。

在spark里，特定的操作需要数据不垮分区分布。在计算期间，一个任务在一个分区上执行，为了所有数据都在单个`reduceByKey`的reduce任务上运行，我们需要执行一个**all-to-all**的操作。它必须从所有分区读取所有的key和key对应的所有的值，并且跨分区聚集去计算每个key的结果，这个过程叫shuffle。

尽管每个分区新shuffle的数据集将是确定的，分区本身的顺序也是这样，但是这些数据的顺序是不确定的。如果希望shuffle后的数据是有序的，可以使用：

- **mapPartitions**对每个分区进行排序，例如**.sorted**。

- **repartitionAndSortWithinPartitions**在分区的同时对分区进行高效的排序。

- **sortBy**做一个整体的排序。

触发**shuffle**的操作包括**repartition**操作，如**repartition**、**coalesce**；**ByKey**操作（除了**counting**相关操作），如**groupByKey**、**reduceByKey**和**join**操作，如**cogroup**、**join**。
### 性能影响

Shuffle是一个代价比较高的操作，它涉及磁盘IO、数据序列化、网络IO。为了准备shuffle操作的数据，spark启动了一系列的map任务和reduce任务，map任务组织数据，reduce完成数据的聚合。这里的map、reduce来自MapReduce，跟Spark的map操作和reduce操作没有关系。

在内部，一个map任务的所有结果会保存在内村，直到内存不能全部存储为止。然后，这些数据将基于目标分区进行排序并写入一个单独的文件中。在reduce时，任务将读取相关已排序的数据块。

某些shuffle操作会大量消耗堆内存空间，因为shuffle操作在数据转换前后，需要在使用内存中的数据结构对数据进行组织。需要特别说明的是，reduceByKey和aggregateByKey在map时会创建这些数据结构，ByKey 操作在reduce时创建这些数据结构。当内存满时，Spark会把溢出的数据存到磁盘上，这将导致额外的磁盘IO开销和垃圾回收开销增加。

shuffle操作还会在磁盘上生成大量中间文件。在Spark 1.3中，这些文件将保留至对应RDD不在使用并被垃圾回收为止。这么做的好处是，如果在Spark重新计算RDD的血统关系时，shuffle操作产生的这些中间文件不需要重新创建。如果Spark应用长期保持对RDD的引用，或者垃圾回收不频繁，这将导致垃圾回收的周期比较长。这意味着，长期运行Spark任务可能会消耗大量的磁盘空间。临时数据存储路径可以通过SparkContext中设置参数spark.local.dir进行配置。