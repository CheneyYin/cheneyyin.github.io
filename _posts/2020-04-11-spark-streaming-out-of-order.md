---
layout: post
title: "Spark Streaming乱序案例"
subtitle: "分析因调度器配置引起的数据同步乱序"
date: 2020-04-11
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags:
 - Spark
 - Spark Streaming
 - Kafka
 - 乱序
 - 公平调度
 - FIFO调度
 - Exactly-Once
 - 累加器
---

# Spark Streaming RDD乱序

## 问题描述

Spark Streaming RDD乱序是指，在其开启**公平调度**的前提下，Spark Streaming周期性地提交Job，并且Job可并发执行。然而在严格的传输语义的要求下（如**Exactly-Once**），Spark Streaming需要向上游系统发送应答记录数据拉取进度，例如**Kafka Consumer**会向**Kafka broker**提交消息偏移位。但是，并发执行的多个Job其处理逻辑虽然相同，但是每个Job的执行时间是难以准确预测。

- 当后继RDD的Job先于前驱RDD完成时，**Kafka broker**记录了后继RDD的偏移位，如果此刻流式应用出现故障，重启应用会从向**Kafka**最后提交的偏移位拉取数据，而完成的前继RDD Job则被忽略，造成数据缺失（当然，每个RDD先做checkpoint是可以解问题的）。
- 当后继RDD的Job先于前驱RDD完成时，**Kafka broker**记录了后继RDD的偏移位，然后先驱RDD的Job也执行完毕，**Kafka broker**记录了先驱RDD的偏移位，那么**Kafka broker**会无视后继RDD完成处理的数据，认为最新的消费进度为先驱RDD提交的偏移位，如果此刻发生故障，应用重新从**Kafka**拉取数据，后继RDD完成处理过的数据会被再次消费，无法做到**Exactly-Once**。

## 场景复现

如下代码可以复现该现象：

```java
Map<String, Object> kafkaParams = new HashMap<>();
Set<String> topics = new HashSet<>();
String bootStrapServer = "192.168.32.61:9092";
String testTopic = "test-data";
String directory = "/home/yanchengyu/kafka-dump";
kafkaParams.put("bootstrap.servers", bootStrapServer);
kafkaParams.put("auto.offset.reset", "earliest");
kafkaParams.put("group.id", "Kafka Debuger");
kafkaParams.put("enable.auto.commit", "false");
kafkaParams.put("session.timeout.ms", "90000");
kafkaParams.put("request.timeout.ms", "95000");
kafkaParams.put("fetch.max.wait.ms", "90000");
kafkaParams.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
kafkaParams.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
topics.add(testTopic);

SparkConf conf = new SparkConf().setMaster("local[*]").setAppName("Kafka Debuger");
conf.set("spark.scheduler.allocation.file", "/home/yanchengyu/spark/fair.xml");
conf.set("spark.scheduler.mode", "FAIR");
conf.set("spark.streaming.concurrentJobs", "4");
JavaStreamingContext ssc = new JavaStreamingContext(conf, Durations.seconds(5));
ssc.sparkContext().setLocalProperty("spark.scheduler.pool", "production");
JavaInputDStream<ConsumerRecord<String, String>> stream =  KafkaUtils.createDirectStream(ssc, 
        LocationStrategies.PreferConsistent(), 
        ConsumerStrategies.Subscribe(topics, kafkaParams));

LongAccumulator acc = ssc.sparkContext().sc().longAccumulator();
acc.setValue(3);

List<TopicPartition> topicPartitions = new ArrayList<>();
topicPartitions.add(new TopicPartition(testTopic, 0));

stream.foreachRDD(rdd -> {

    HasOffsetRanges hors = (HasOffsetRanges)(rdd.rdd());
    OffsetRange[] ranges = hors.offsetRanges();
    int rddId = rdd.id();
    rdd.foreachPartition(iter -> {
        int partitionId = TaskContext.getPartitionId();
        String partitionFile = String.format("%d_%d_%d-%d", 
                rddId, partitionId, ranges[partitionId].fromOffset(), ranges[partitionId].untilOffset());
        File dumpFile = new File(directory + "/" + partitionFile);
        if (dumpFile.exists()) {
            dumpFile.delete();
        } else {
            dumpFile.createNewFile();
        }

        BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(new FileOutputStream(dumpFile)));
        while(iter.hasNext()) {
            ConsumerRecord<String, String> record = iter.next();
            writer.write(record.key() + "\t" + record.value() + "\n");
        }
        writer.flush();
        writer.close();

    });

    if (rdd.id() > 3) {
        return ;
    }
    while(rddId != acc.value()) {;}

    CanCommitOffsets committer = (CanCommitOffsets)stream.inputDStream();
    committer.commitAsync(ranges, new OffsetCommitCallback() {

        @Override
        public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception exception) {
            // TODO Auto-generated method stub
            for (TopicPartition topicPartition : topicPartitions) {

                long offset = offsets.get(topicPartition).offset();
                String marker = String.format("%d_%d_%d_-%d", 
                        System.currentTimeMillis(), rddId, topicPartition.partition(), offset);
                File offsetMarker = new File(directory + "/" + marker);
                if (offsetMarker.exists()) {
                    offsetMarker.delete();
                } else {
                    try {
                        offsetMarker.createNewFile();
                    } catch (IOException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }
                }	
            }
            acc.add(-1);
        }
    });
});
ssc.start();
try {
    ssc.awaitTermination();
} catch (InterruptedException e) {
    // TODO Auto-generated catch block
    e.printStackTrace();
}
```

文件目录如下，

```shell
-rw-r--r--  1 root       root       648815 1月  11 10:32 0_0_442728-477283
-rw-r--r--  1 root       root       331816 1月  11 10:32 1_0_477283-494952
-rw-r--r--  1 root       root            0 1月  11 10:32 1547173970001_3_0_-547959
-rw-r--r--  1 root       root            0 1月  11 10:32 1547173975001_2_0_-530290
-rw-r--r--  1 root       root            0 1月  11 10:33 1547173980001_1_0_-494952
-rw-r--r--  1 root       root            0 1月  11 10:33 1547173985002_0_0_-477283
-rw-r--r--  1 root       root       663632 1月  11 10:32 2_0_494952-530290
-rw-r--r--  1 root       root       331816 1月  11 10:32 3_0_530290-547959
-rw-r--r--  1 root       root       663632 1月  11 10:32 4_0_547959-583297
-rw-r--r--  1 root       root            0 1月  11 10:32 5_0_583297-583297
-rw-r--r--  1 root       root            0 1月  11 10:32 6_0_583297-583297
-rw-r--r--  1 root       root            0 1月  11 10:33 7_0_583297-583297
-rw-r--r--  1 root       root            0 1月  11 10:33 8_0_583297-583297
-rw-r--r--  1 root       root            0 1月  11 10:33 9_0_583297-583297
```

目录包含两类文件，`1_0_477283-494952`这类为写的数据文件，`1`表示RDD的ID，`0`表示分区号，`477283-494952`表示对应**Kafka**数据偏移位范围为$ [477283,494952) $。`1547173970001_3_0_-547959`为已提交的消费进度信息，`1547173970001`为时间戳，`3`表示RDD的ID，`0`表示分区号，  `547959`提交的偏移位。

从目录可以看出，**Kafka broker**记录的偏移位依次来自RDD 3、RDD 2、RDD 1、RDD 0。如果发生故障，应用重启后，会再次消费$ [477283, 547959) $的数据。

## 解决方法

- 按照RDD的生成顺序来提交偏移位，是可以解决**Kafka broker**偏移保持和实际消费场景(Story)不一致的现象。使用累加器保序是一种实现方式。

  ```java
  JavaInputDStream<ConsumerRecord<String, String>> stream =  KafkaUtils.createDirectStream(ssc, 
          LocationStrategies.PreferConsistent(), 
          ConsumerStrategies.Subscribe(topics, kafkaParams));
  
  LongAccumulator acc = ssc.sparkContext().sc().longAccumulator();
  acc.setValue(0);
  
  List<TopicPartition> topicPartitions = new ArrayList<>();
  topicPartitions.add(new TopicPartition(testTopic, 0));
  
  stream.foreachRDD(rdd -> {
  
      HasOffsetRanges hors = (HasOffsetRanges)(rdd.rdd());
      OffsetRange[] ranges = hors.offsetRanges();
      int rddId = rdd.id();
      rdd.foreachPartition(iter -> {
          int partitionId = TaskContext.getPartitionId();
          String partitionFile = String.format("%d_%d_%d-%d", 
                  rddId, partitionId, ranges[partitionId].fromOffset(), ranges[partitionId].untilOffset());
          File dumpFile = new File(directory + "/" + partitionFile);
          if (dumpFile.exists()) {
              dumpFile.delete();
          } else {
              dumpFile.createNewFile();
          }
  
          BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(new FileOutputStream(dumpFile)));
          while(iter.hasNext()) {
              ConsumerRecord<String, String> record = iter.next();
              writer.write(record.key() + "\t" + record.value() + "\n");
          }
          writer.flush();
          writer.close();
  
      });
      while(rddId != acc.value()) {;}
  
      CanCommitOffsets committer = (CanCommitOffsets)stream.inputDStream();
      committer.commitAsync(ranges, new OffsetCommitCallback() {
  
          @Override
          public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception exception) {
              // TODO Auto-generated method stub
              for (TopicPartition topicPartition : topicPartitions) {
  
                  long offset = offsets.get(topicPartition).offset();
                  String marker = String.format("%d_%d_%d_-%d", 
                          System.currentTimeMillis(), rddId, topicPartition.partition(), offset);
                  File offsetMarker = new File(directory + "/" + marker);
                  if (offsetMarker.exists()) {
                      offsetMarker.delete();
                  } else {
                      try {
                          offsetMarker.createNewFile();
                      } catch (IOException e) {
                          // TODO Auto-generated catch block
                          e.printStackTrace();
                      }
                  }	
              }
              acc.add(1);
          }
      });
  });
  ssc.start();
  try {
      ssc.awaitTermination();
  } catch (InterruptedException e) {
      // TODO Auto-generated catch block
      e.printStackTrace();
  }
  ```

  这种方法保证了并发量，但是，会因为等待提交偏移位而浪费计算资源，并且频繁地访问累加器会增加计算和网络传输开销。
- 为流式应用配置**FIFO**调度器。这种做法虽然不会浪费计算资源，但并发量会受限制，处理速率有降低的可能。