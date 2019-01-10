---
title:  "Spark优化"
layout: archive
date:   2018-12-23 08:44:13
categories: spark
---

### 钨丝计划(Project Tungsten)

#### Spark极限压榨硬件性能

- 钨丝计划是Spark项目自启动以来，有史以来最大的变化。

- 它着重于大幅度提高用于Spark应用程序的内存和CPU的效率，从而将性能推向现代硬件的极限。

##### 钨丝计划主要包括以下三个方面

1. 内存管理和二进制处理
   - 利用应用程序语义显式地管理内存，消除JVM对象模型和垃圾收集的开销
2. 缓存友好的计算
   - 利用内存层次结构的算法和数据结构
3. 代码生成
   - 使用代码生成来利用现代编译器和cpu

Spark1.3改进

1. `Java RDD`、`Python RDD`、`R RDD`
2. `Scala RDD` (高阶函数， 隐式特性)
3. `Spark SQL`(Catalyst)

**对CPU效率的关注源于这样一个事实，即Spark工作负载越来越多地受到CPU和内存使用的限制，而不是IO和网络通信的限制**。

为什么CPU成为新的瓶颈？

> 一是硬件配置提供了越来越大的IO总带宽，比如网络中的10Gbps链路和用于存储的高带宽SSD或条纹硬盘阵列。
>
> 从软件的角度来看，Spark的优化器现在允许许多工作负载通过修剪给定作业中不需要的输入数据来避免重要的磁盘IO。
>
> 在Spark的shuffle子系统中，串行化和散列(受CPU限制)已经被证明是关键瓶颈，而不是底层硬件的原始网络吞吐量。
>
> **Spark工作负载越来越多的受到Cpu和内存使用的限制，而不是IO和网络的限制**。

###  一、数据序列化

#### 1. 内存管理和二进制处理

- JVM上的应用程序通常依赖JVM的垃圾收集器来管理内存。

JVM存在的问题

1. JVM为了达到数据的通用性，通常会把UTF-8的数据在JVM层面，转换为编码为UTF-16的形式，此外还会为每种数据类型加上12字节的header和8个字节的hashcode，总的字节总量会比原始内容的字节数量超出12倍左右。**在JVM对象模型中，一个简单的4字节字符串总共超过48字节!**

2. JVM垃圾回收。

![img](https://xlactive-1258062314.cos.ap-chengdu.myqcloud.com/2018-12-25%209-11-53.JPG)

- 新生代：主要用于存放瞬时变换的对象（生命周期短暂）
- 老年代：主要存放需要长期使用的对象（static)

**JVM内存存储过程：**

1. 首先会把对象存在Eden的内存区域
2. 会将Eden内存中的数据转移到Sur1
3. 当Sur1中的内存不足时，就转移到Sur2
4. Sur2的内存不够时，就会将Sur2中的数据转移到老年代

一旦转移到老年代，老年代中的数据，是不会被GC(回收)掉的

此时，出现一个**OOM（out of memory）**其中的一个原因：

- 老年代内存不够用了

**GC（垃圾回收）算法**

- JVM中的GC回收，通常发生在新生代。

GC回收：

- 检测对象不可达（查看某一个Java对象是否还在引用）

当有异常**GC stop world 或者 read time out**此时应注意

- 尽量少的使用static对象或static类
- 尽量避免使用递归算法
- 尽量避免使用 `+`号进行大批字符串拼接

#### 2. 缓存友好计算

除了主机的内存之外，还利用了CPU上的三级缓存来进行计算的提速。

速度： `1级缓存 > 2级缓存  > 3级缓存 > 内存`

Spark是众所周知的内存计算引擎

- Spark可以有效利用集群上的内存资源，以比基于磁盘的解决方案高得多的速度处理数据
- Spark还可以处理比可用内存大几个数量级的数据，透明地溢出到磁盘，并执行排序和散列等外部操作
- 缓存感知计算通过更有效地使用`L1/ L2/L3` CPU缓存来提高数据处理的速度，因为它们比主内存快几个数量级

这如何适用于Spark?大多数分布式数据处理可以归结为一个小的操作列表

例如聚合、排序和连接。通过提高这些操作的效率，我们可以提高Spark应用程序的整体效率。

#### 3. 代码生成

- Spark引入了用于SQL和DataFrames表达式计算的代码生成。

Hive Catalyst(SQL优化器)

### 二、数据序列化

Spark提供两种序列化类库：

1. Java serialization(Spark默认)
   - Java序列化是灵活的，但通常很慢，并导致许多类的大型序列化格式。不建议使用

2. Kryo serialization
   - Kryo比Java序列化快得多，也更紧凑(通常是10倍)，但不支持所有的类型，使用前要先注册。

kryo serialization注册:

`conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")`

要注册`Kryo`中注册自己的类，需要使用`registerKryoClasses` method。

```scala
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))
val sc = new SparkContext(conf)
```

如果你的对象很大，你可以设置`spark.kryoserializer.buffer`。

如果不注册自己的类，`kroy`可以正常运行，但是它必须为每个对象存在完整类名，这是一种浪费。

