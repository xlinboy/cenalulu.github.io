---
layout: post
title: '[Mapreduce]执行流程'
subtitle: 'MapReduce框架结构以及核心机制'
date: 2018-12-16
categories: Hadoop
cover: '../../../assets/img/hadoop-logo.jpg'
tags: Hadoop
---

> MapReduce是一个分布式运算程序的编程框架。其核心功能是：将用户编写的业务逻辑代码和自带默认组件整合成一个完整的分布式运算程式。

### MapReduce框架结构以及核心机制
一个完整的mapreduce程序在分布式运行时有三类实例进程：
1. MRAppMaster：负责整个程序的过程调度及状态协调
2. MapTask：负责map阶段整个数据处理流程
3. ReduceTask：负责reduce阶段的整个数据处理流程

流程图

![mark](https://xlactive-1258062314.cos.ap-chengdu.myqcloud.com/dbfjHdHKiG.png)  
#### 流程分析
1. 一个mr程序启动的时候，最先启动的是MRAppMaster，MRAppMaster启动后根据本次job的描述信息，计算出需要的maptask实例数量，然后向集群申请机器启动相应数量的maptask进程
2. maptask进程启动之后，根据给定的数据切片范围进行数据处理，主体流程为：
    1. 利用客户指定的inputformat来获取RecordReader读取数据，形成输入<K,V>对 
    2. 将输入KV对传递给客户定义的map()方法，做逻辑运算，并将map()方法输出的KV对收集到缓存
    3. 将缓存中的KV对按照K分区排序后不断溢写到磁盘文件
3. MRAppMaster监控到所有maptask进程任务完成之后，会根据客户指定的参数启动相应数量的reducetask进程，并告知reducetask进程要处理的数据范围（数据分区）
4. Reducetask进程启动之后，根据MRAppMaster告知的待处理数据所在位置，从若干台maptask运行所在机器上获取到若干个maptask输出结果文件，并在本地进行重新归并排序，然后按照相同key的KV为一个组，调用客户定义的reduce()方法进行逻辑运算，并收集运算输出的结果KV，然后调用客户指定的outputformat将结果数据输出到外部存储

### MapTask并行度的决定机制

> 一个job的map阶段并行度由客户端在提交job时决定

客户端对map阶段并行度的规划的基本逻辑为:  

- 将待处理数据执行逻辑切片（即按照一个特定切片大小，将待处理数据划分成逻辑上的多个split），然后**每一个split分配一个mapTask并行实例处理**。  

逻辑切片规划描述文件，由FileInputFormat实现类的getSplits()方法完成：  

![mark](https://xlactive-1258062314.cos.ap-chengdu.myqcloud.com/3Fb2Fm7Fb3.png)  

#### FileInputFormat切片机制

1. 切片定义在InputFormat类中的getSpilt()方法
2. FileInputFormat中默认的切片机制：
   - 简单地按照文件的内容长度进行切片  
   - 切片大小，默认等于block大小 
   - 切片时不考虑数据集整体，而是逐个针对每一个文件单独切片   

比如有两个待处理文件
```
file1.txt    320M
file2.txt    10M
```
经过FileInputFormat的切片机制运算后，形成的切片信息如下：
```
file1.txt.split1--  0~128
file1.txt.split2--  128~256
file1.txt.split3--  256~320
file2.txt.split1--  0~10M
```

3. FileInputFormat中切片的大小参数配置

计算切片大小逻辑：Math.max(minSize, Math.min(maxSize, blockSize))

```
minsize：默认值：1  
  	配置参数： mapreduce.input.fileinputformat.split.minsize 
maxsize：默认值：Long.MAXValue  
    配置参数：mapreduce.input.fileinputformat.split.maxsize
 
blocksize
```
**默认情况下，切片大小=blocksize**。如果最后一块大小小于切片大小的1.1倍，会放在同一个块

### MapReduce中的Combiner
Combiner的使用要非常谨慎  

因为combiner在mapreduce过程中可能调用也肯能不调用，可能调一次也可能调多次

所以：combiner使用的原则是：有或没有都不能影响业务逻辑

#### Combiner简介
1. combiner是MR程序中Mapper和Reducer之外的一种组件
2. combiner组件的父类就是Reducer
3. combiner和reducer的区别在于运行的位置：  
    - Combiner是在每一个maptask所在的节点运行;   
    - Combiner是在每一个maptask所在的节点运行;
4. combiner的意义就是对每一个maptask的输出进行局部汇总，以减小网络传输量
5. combiner能够应用的前提是不能影响最终的业务逻辑

combiner的输出kv应该跟reducer的输入kv类型要对应起来

#### 具体实现步骤
1. 自定义一个combiner继承Reducer，重写reduce方法
2. 在job中设置：  job.setCombinerClass(CustomCombiner.class)

#### MapReduce的shuffle机制

- mapreduce中，map阶段处理的数据如何传递给reduce阶段，是mapreduce框架中最关键的一个流程，这个流程就叫shuffle。 
- shuffle：洗牌、发版、混洗--核心机制：数据分区，排序，缓存。   
- 具体来说：就是将maptask输出的处理结果数据，分发给reducetask，并在分发的过程中，对数据按key进行了分区和排序。

详细过程

![mark](https://xlactive-1258062314.cos.ap-chengdu.myqcloud.com/6C2kDE0946.png)  
1. maptask收集我们的map()方法输出的<k,v>对，放到缓形缓冲区（数组）。
2. 从内存缓冲区不断溢出本地磁盘（环形缓冲区需要进行排序，以不会写满在溢出），可能会溢出多个文件。
3. 多个溢出文件会被合并成大的溢出文件。
4. 在溢出过程中，及合并的过程中，都要调用partitoner进行分组和针对key进行排序。
5. reducetask根据自己的分区号，去各个maptask机器上取相应的结果分区数。
6. reducetask会取到同一个分区的来自不同maptask的结果文件，reducetask会将这些文件再进行合并（归并排序）。
7. 合并成大文件后，shuffle的过程也就结束了，后面进入reducetask的逻辑运算过程（从文件中取出一个一个的键值对group，调用用户自定义的reduce()方法）。

Shuffle中的缓冲区大小会影响到mapreduce程序的执行效率，原则上说，缓冲区越大，磁盘io的次数越少，执行速度就越快。    
缓冲区的大小可以通过参数调整,  参数：io.sort.mb  默认100M

### MapReduce中有序列化

Java的序列化太冗余不好，hadoope有一套自己的序列化机制（Writable）

#### 自定义序列化接口

- 不需要比较只需要继承Writable即可
- 如果需要将自定义的bean放在key中传输，则还需要实现comparable接口，因为**mapreduce框中的shuffle过程一定会对key进行排序**,此时，自定义的bean实现的接口应该是：
`public  class  FlowBean  implements  WritableComparable<FlowBean> `

```
/**
	 * 反序列化的方法，反序列化时，从流中读取到的各个字段的顺序应该与序列化时写出去的顺序保持一致
	 */
	@Override
	public void readFields(DataInput in) throws IOException {
		upflow = in.readLong();
		dflow = in.readLong();
		sumflow = in.readLong();
	}
	/**
	 * 序列化的方法
	 */
	@Override
	public void write(DataOutput out) throws IOException {
		out.writeLong(upflow);
		out.writeLong(dflow);
		//可以考虑不序列化总流量，因为总流量是可以通过上行流量和下行流量计算出来的
		out.writeLong(sumflow);

	}
	
	@Override
	public int compareTo(FlowBean o) {
		
		//实现按照sumflow的大小倒序排序
		return sumflow>o.getSumflow()?-1:1;
	}

```

### Mapreduce中的分区Partitioner

Mapreduce中会将map输出的kv对，按照相同key分组，然后分发给不同的reducetask    

**默认的分发规则为：根据key的hashcode%reducetask数来分发**。 
自定义Partitioner类需要继承抽象类CustomPartitioner 
然后在job对象中，设置自定义partitioner： job.setPartitionerClass(CustomPartitioner.class)  