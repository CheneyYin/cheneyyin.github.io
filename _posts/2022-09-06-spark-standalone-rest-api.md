---
layout: post
title: "Spark Standalone集群隐藏的REST API"
subtitle: "介绍如何使用Spark Standalone隐藏的REST API"
date: 2022-09-06
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags:
 - Spark
 - Spark Standalone
 - REST API
---

# Spark Standalone集群隐藏的REST API

## 配置

在`spark-defaults.conf`中配置，

```shell
spark.master.rest.enabled true
```

## API

### 提交

使用`/v1/submissions/create`去提交任务，

```she
curl -X POST http://master-0.spark.node:6066/v1/submissions/create --header "Content-Type:application/json;charset=UTF-8" --data '
{
  "appResource": "/home/spark/spark/examples/jars/spark-examples_2.12-3.3.0.jar",
  "sparkProperties": {
    "spark.master": "spark://master-0.spark.node:7077,master-1.spark.node:7077",
    "spark.app.name": "Spark REST API - PI",
    "spark.submit.deployMode": "client",
    "spark.eventLog.enabled": "true",
    "spark.eventLog.dir": "hdfs://namenode:9000/shared/spark-logs",
    "spark.jars": "/home/spark/spark/examples/jars/spark-examples_2.12-3.3.0.jar"
  },
  "clientSparkVersion": "3.3.0",
  "mainClass": "org.apache.spark.examples.SparkPi",
  "environmentVariables": {
    "SPARK_ENV_LOADED": "1"
  },
  "action": "CreateSubmissionRequest",
  "appArgs": [
    "1000"
  ]
}
'
```

提交成功会返回如下结果，

```json
{
  "action" : "CreateSubmissionResponse",
  "message" : "Driver successfully submitted as driver-20220718051203-0008",
  "serverSparkVersion" : "3.3.0",
  "submissionId" : "driver-20220718051203-0008",
  "success" : true
}
```

### 查看

使用`/v1/submissions/status`去查看任务状态，在`status`后加入`submissionId`，

```she
curl http://master-0.spark.node:6066/v1/submissions/status/driver-20220718051203-0008
```

返回如下，

```json
{
  "action" : "SubmissionStatusResponse",
  "driverState" : "RUNNING",
  "serverSparkVersion" : "3.3.0",
  "submissionId" : "driver-20220718051203-0008",
  "success" : true,
  "workerHostPort" : "192.168.8.101:44977",
  "workerId" : "worker-20220718031455-192.168.8.101-44977"
}
```

### kill

使用`/v1/submissions/kill`去关闭任务，仍然使用`submissionId`为参数，

```shell
curl -X POST http://master-0.spark.node:6066/v1/submissions/kill/driver-20220718051203-0008
```

返回，

```json
{
  "action" : "KillSubmissionResponse",
  "message" : "Kill request for driver-20220718051203-0008 submitted",
  "serverSparkVersion" : "3.3.0",
  "submissionId" : "driver-20220718051203-0008",
  "success" : true
}
```

