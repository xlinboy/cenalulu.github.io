---
layout: post
title: '[Flume-Spark]整合'
subtitle: 'Flume+Spark整合案例'
date: 2018-12-16
categories: Spark
cover: '../../../assets/img/spark.png'
tags: Spark
---

### Spark Streaming + Flume

有两个整合方式

- Flume将数据Push（推） 到 Sparking Streaming
- Sparking Streaming 从Flume pull（拉）数据

#### 1 Push

需要先启动Spark Streaming 程序，然后启动Flume

Flume 配置

```shell
a1.sources = r1
a1.sinks = k1
a1.channels = c1
# sources
a1.sources.r1.type = netcat
a1.sources.r1.bind = hadoop1
a1.sources.r1.port = 12345
# sinks
a1.sinks.k1.type = avro
a1.sinks.k1.hostname = hadoop1
a1.sinks.k1.port = 44444
# channels
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

依赖的Jar包

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.10</artifactId>
    <version>1.6.3</version>
</dependency>
```

Spark Streaming 程序

```scala
val conf = new SparkConf()
      .setAppName("sparkStreaming")
      .setMaster("local[2]")
# 控制台输出的日志太多，不方便测试，将日志输出级别降低
Logger.getLogger("org.apache.spark").setLevel(Level.ERROR)
Logger.getLogger("org.eclipse.jetty.server").setLevel(Level.OFF)

val ssc = new StreamingContext(conf, Seconds(5))

val flumeStream = FlumeUtils.createStream(ssc, "hadoop1", 44444)

flumeStream.map(x => new String(x.event.getBody.array()).trim)
    .flatMap(_.split(",")).map((_, 1))
    .reduceByKey(_+_)
    .print()

ssc.start()
ssc.awaitTermination()
```

#### 2 Pull (推荐使用)

Pull支持事务，可以提高容错性。

Flume 配置

```shell
a1.sources = r1
a1.sinks = k1
a1.channels = c1
# sources
a1.sources.r1.type = netcat
a1.sources.r1.bind = hadoop1
a1.sources.r1.port = 12345
# sinks
a1.sinks.k1.type = org.apache.spark.streaming.flume.sink.SparkSink
a1.sinks.k1.hostname = hadoop1
a1.sinks.k1.port = 44444
a1.sinks.k1.channel = memoryChannel
# channels
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

依赖Jar包

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming-flume-sink_2.10</artifactId>
    <version>1.6.3</version>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.3.2</version>
</dependency>
<dependency>
    <groupId>org.scala-lang</groupId>
    <artifactId>scala-library</artifactId>
    <version>2.10.5</version>
</dependency>
```

Spark Streaming 代码

```scala
val ssc = new StreamingContext(conf, Seconds(5))
val flumeStream = FlumeUtils.createPollingStream(ssc, "hadoop1", 44444)

flumeStream.map(x => new String(x.event.getBody.array()).trim)
    .flatMap(_.split(",")).map((_, 1))
    .reduceByKey(_+_)
    .print()

ssc.start()
ssc.awaitTermination()
```



参考：

[Spark Streaming + Flume Integration Guide](http://spark.apache.org/docs/1.6.3/streaming-flume-integration.html)