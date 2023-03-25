---
layout: post
title: "Spark Standalone集群启动配置"
subtitle: "介绍Spark Standalone集群启动的常用配置项目"
date: 2022-08-29
author: "Cheney.Yin"
header-img: "//imgloc.com/i/iADI63"
tags:
 - Spark
 - Spark Standalone
---

# Spark Standalone集群启动配置

| 配置项                      | 含义                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `-h HOST`, `--host HOST`    | 监听Host Name                                                |
| `-p PORT`,`--port PORT`     | 监听端口（默认：Master使用7077，worker随机设置端口）         |
| `--webui-port PORT`         | Web UI的端口（默认：Master使用8080，worker使用8081）         |
| `-c CORES`，`--cores CORES` | **Worker专属配置**，可分配给Spark应用使用的CPU总数量。（默认：主机上的全部CPU均可以使用。） |
| `-m MEM`，`--memory MEM`    | **Worker专属配置**，可分配给Spark应用使用的内存量，例如配置为`1000M`或`2G`。（默认：主机的全部内存，最小为1GiB。） |
| `-d DIR`，`--work-dir DIR`  | **Worker专属配置**，暂存空间和Job输出日志目录。（默认：`SPARK_HOME/work`。） |
| `--properties-file FILE`    | Spark配置文件路径。（默认：`conf/spark-defaults.conf`）      |

