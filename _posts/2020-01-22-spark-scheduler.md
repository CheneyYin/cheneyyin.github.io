---
layout: post
title: "Spark资源调度"
subtitle: "介绍Spark跨应用调度和应用内调度的相关概念和配置"
date: 2020-01-22
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags:
 - Spark
 - Scheduler
 - 公平调度
 - FIFO调度
 - Executor
---

# 概况

spark 具备几种计算资源的调度机制。首先，在集群模式下，每个Spark应用(SparkContext的实例)运行在一组独立的执行器(executor)进程中。Spark集群管理器提供了跨应用的调度机制。其次，在每个Spark应用中，如果多个job(Spark actions)被不同的线程提交，它们能够并发执行。

# 跨应用调度

运行在spark集群的应用，可以获得一组独立的executor JVM，这些JVM只运行应用下发的任务、存储应用数据。如果多个用户需要共享使用集群，它们的资源分配管理选项可以不同，并且依赖集群管理器。

最简单的选项是资源静态分区(static partitioning)，它在所有集群管理器上都是有效的。每个应用都要设定一个生命周期内使用和持有资源的最大值。Spark‘s standalone、YARN 和 coarse-grained Mesos mode都支持这种资源分配策略。不同的集群模式下，资源配置如下：

- Standalone mode: 默认情况下，应用提交至standalone 的集群后，将进入FIFO队列等待调度，并且应用会尽可能使用全部有效的计算节点。

  |      | 配置项                    | 说明                   |
  | ---- | ------------------------- | ---------------------- |
  | CPU  | spark.cores.max           | 应用使用的core数量上限 |
  | CPU  | spark.deploy.defaultCores | 应用默认使用的core数量 |
  | 内存 | spark.executor.memory     | executor内存大小       |

- Mesos: 为了在Mesos上使用静态分区，可配置如下内容。

  |      | 配置项                | 说明                     |
  | ---- | --------------------- | ------------------------ |
  | 启用 | spark.mesos.coarse    | 值为true，启用静态分区。 |
  | CPU  | spark.cores.max       | 应用使用的core数量上限   |
  | 内存 | spark.executor.memory | executor内存大小         |

- YARN: 可以由Spark Yarn client控制集群分配的执行器数量，配置内容如下。

  |      | Spark YARN Client配置项 | 集群配置项               | 说明               |
  | ---- | ----------------------- | ------------------------ | ------------------ |
  | CPU  | --num-executors         | spark.executor.instances | 控制执行器数量     |
  | CPU  | --executor-cores        | spark.executor.cores     | 每个执行器分配核数 |
  | 内存 | --executor-memory       | spark.executor.memory    | 每个执行器内存量   |

  跨应用的内存不可共享使用，CPU核实动态共享使用的。

  # 动态资源分配

  Spark提供了一种动态资源调度机制，根据工作负载去调整应用占用的资源。这意味着，应用可以释放不使用的资源，当应用需要时可以再次申请。在多应用共享资源的Spark集群中，这种特性是十分有用的。
  
  该机制默认是关闭的，所有coarse-gained集群管理器都可以启动动态资源调度（standalone、YARN、mesos coarse-gained）。

  ## 配置和安装

  开启动态资源调度，需要配置两项。首先，应用要配置spark.dynamicAllocation.enabled为true；然后，每个worker都要开启external shuffle service， 应用需要配置spark.shuffle.service.enabled为true。
  
  其它与之相关的配置项，都在spark.dynamicAllocation.\*和spark.shuffle.service.\*命名空间中。

  ## 资源分配策略

  动态资源分配，应该能使Spark释放不再使用的执行器，为需要资源的应用分配执行器。由于没有明确的方式去预测将要被移除的执行器未来是否会执行任务，将要新添的执行器未来是否会空闲。所以需要一些启发因子来确定执行器何时被删除和添加。

  ### 请求策略

  开启动态分配的Spark应用，在它提交的任务还处于挂起等待状态时，应用将申请额外的执行器。这种场景其实暗示了现存的执行器不足以同时覆盖提交执行未完的任务。
  	
  Spark以轮次方式请求执行器。拥有挂起任务的应用，会在spark.dynamicAllocation.schedulerBacklogTimeout秒后，发起请求；并且每隔spark.dynamicAllocation.sustainedSchedulerBacklogTimeout秒，应用将请求一次。每轮请求的执行器数量将指数式的增长。例如，某个应用在第一轮增加了一个执行器，随后将增加2、8、16.....个执行器。

  ### 删除策略

  删除策略十分简单。Spark应用会将空闲时间超过spark.dynamicAllocation.executorIdleTimeout秒的执行器移除。在多数情况下，该条件和请求条件互斥，如果应用还存在挂起任务，执行器不会空闲。

  ## 解除执行器

  未启用动态分配时，一个Spark 执行器出现失败后，或者应用退出时，执行器会退出。在这两种场景下，所有和执行器关联的状态不再需要，可以被安全丢弃。在动态分配下，当执行器移除后，应用仍会运转。如果应用试图去访问存放在执行器的状态，则需要重新计算状态。所以，Spark需要一套执行器解除机制，在执行器清理前将状态保存。
  	
  这种需求对shuffle是非常重要的。在一次shuffle过程中，Spark执行器将其map输出结果写入本地磁盘文件，其它执行器试图去拉取这些文件，该执行器会为其它执行器提供文件访问服务。 然而，当出现掉队现象时，一些任务和其它同代任务相比，未能及时去拉取shuffle文件，导致动态分配在shuffle真正完成前删除执行器，进而使得部分shuffle文件被重计算。
  	
  Spark 1.2版本开始使用external shuffle service保存shuffle文件。在集群的每个节点上都会运行external shuffle service， external shuffle service是一个long-running 进程，并且独立于spark应用和应用相关的执行器。如果external shuffle service生效，spark执行器会从shuffle service拉取文件，替代从其它执行器获取。这意味着，任何执行器写出的shuffle状态超出了执行器的生命周期。
  	
  除shuffle文件之外，执行器也会在磁盘和内存中缓存数据。当执行器被删除后，所有缓存数据将不能访问。为了缓解该问题，可以默认执行器缓存的数据不被删除。你可以通过spark.dynamicAllocation.cachedExecutorIdleTimeout配置该行为。在将来的版本，缓存的数据可以保存在堆外存储中，和external shuffle service保存数据的思路类似。

  # Spark应用内部调度

  在一个Spark应用内，多个由不同线程提交的job可以同时并行执行。Spark的action行为会触发生成一个job。
  	
  默认情况下，spark调度器按照FIFO的方式调取job。每个job可分为多个stage，FIFO的调度基本准则是(1).不同job间，优先执行；(2)队头job未使用全部集群资源，后续job可别调取执行。(3).同一job内，先驱stage优先调取执行。
  	
  从spark 0.8开始，Fair调度器被使用。在Fair调度模式下，spark以轮转的方式分配job的任务。这意味着即使长job在运行，短job在提交后仍然可以立即分配资源执行。这种调度模式适合多用户的情况。
  	
  在SparkContext配置spark.scheduler.mode为FAIR，便可开启FAIR调度器。

  ```scala
  val conf = new SparkConf().setMaster(...).setAppName(...)
  conf.set("spark.scheduler.mode", "FAIR")
  val sc = new SparkContext(conf)
  ```

  ## Fair调度池

  Fair调度器支持将job放入调度池，并且为每个子池配置不同的参数。调度池适用于job轻重缓急的情况。如果没有任何干预，新提交的job会进入默认调度池。job的调度池可以在SparkContext中配置spark.scheduler.pool来选择它们需要提交的调度池。

  ```scala
  // Assuming sc is your SparkContext variable
  sc.setLocalProperty("spark.scheduler.pool", "pool1")
  ```

  在配置SparkContext的local property后，该线程内所有提交的job将会使用该调度池的名字。使用完毕后，通过如下命令清理调度池。

  ```scala
  sc.setLocalProperty("spark.scheduler.pool", null)
  ```

  ## 调度池的默认行为

  默认情况下，所有调度池平等的共享集群资源，并且每个调度池内部由FIFO调取job。例如，你为每个用户创建一个调度池，这意味着每个用户将平等地共享集群资源，每个用户的查询会按序进行，而不是后来的查询要等待先前的用户释放资源。

  ## 调度池配置

  每个调度池包含三个属性：

  - schedulingMode: FIFO或FAIR。

  - weight: 调度池之间共享资源的相对权重。

  - minShare: 每个调度池使用的最小CPU核数量。默认值为0.

调度池的配置文件是XML格式，可以参考conf/fairscheduler.xml.template，在SparkConf中设置spark.scheduler.allocation.file即可。

    ```scala
    conf.set("spark.scheduler.allocation.file", "/path/to/file")
    ```

fairscheduler.xml例子如下，

    ```xml
    <?xml version="1.0"?>
    <allocations>
      <pool name="production">
        <schedulingMode>FAIR</schedulingMode>
        <weight>1</weight>
        <minShare>2</minShare>
      </pool>
      <pool name="test">
        <schedulingMode>FIFO</schedulingMode>
        <weight>2</weight>
        <minShare>3</minShare>
      </pool>
    </allocations>
    ```

包含连个子池，分别为production、test。FIFO调度池默认的weight为1、minShare为3。

Fair的调度准则如下，

1. 调度池运行的task数小于minShare，则优先调取。
2. 调度池运行task数/minShare越小，则优先调取。
3. 调度池运行task数/weight越小，则优先调取。
4. 比较名称。

Fair调度器轮转执行，每轮先选择调度池，然后从池中选择任务。

