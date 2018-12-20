---
layout: article
title:  "Hdfs中的SNN的详细流程"
categories: hadoop
date:   2018-12-18 08:44:13
tags: sqoop
description: 'sqoop 是什么呢，适合场景如何？'
---



## NameNode

> 主要用来保存HDFS元数据信息，比如命名空间信息，块信息等。当它运行的时候，这些信息是存在内存中的。但是这些信息也可以持久化到磁盘上。

![mark](http://pc06h57sq.bkt.clouddn.com/blog/181010/6b8FC7bb9f.png?imageslim)  

1. fsimage: 它是在NameNode启动时对整个文件系统的快照
2. edits log: 它是在NameNode启动后，对文件系统的改动序列

&emsp;&emsp;只有在NameNode重启时，edits log才会合并到fsimage文件中，从而得到一个文件系统的最新快照。但是在生产环境集群中的NameNode是很少重启的， **这意味者当NameNode运行来很长时间后，edits文件会变的很大。** 在这种情况下就会出现下面这些问题：
1. edits log文件会变的很大，怎么去管理这个文件是一个挑战。
2. NameNode的重启会花费很长时间，因为有很多改动[在edits log中]要合并到fsimage文件上。
3. 如果NameNode挂掉了，那我们就丢失了很多改动因为此时的fsimage文件非常旧。

## Secondary NameNode
> 主要作用：帮我们减少edits log文件的大小和得到一个最新的fsimage文件，从减少namenode的压力。它的职责是，合并NameNode的edits  log到fsimage文件中

![mark](http://pc06h57sq.bkt.clouddn.com/blog/181016/DF19igj73D.png?imageslim) 

1. 客户端向namenode发出更新元数据的请求
2. snn 向nn请求checkpoint（合并数据）
3. 此时namenode会滚动当前在操作的日志文件edits.inprogress。
4. snn会将edits文件(多个滚动的文件)和镜像文件fsimage下载
5. 根据操作日志文件根据一些算法计算出元数据从而来和镜像文件进行合并存到内存中。
6. 将合并后的元数据dump成新的image文件，持久化到硬盘(fsimage.chkpoint)
7. 上传新的fsimage到NameNode。
8. NameNode把旧的fsimage用新的覆盖掉。


#### 思考题
日志的滚动：  
> 日志文件是一个正在写的edits.inprogress,多个滚动的ecits.000001..。checkpoint时，将正在写的滚动。

SNN的特点：
> Secondary NameNode所做的是在文件系统这设置一个Checkpoint来**帮助NameNode更好的工作**;它不是取代NameNode，也不是NameNode的备份。

Secondary NameNode的检查进程启动，由两个参数配置:
1. fs.checkpoint.period，指定连续两次检查点的最大时间间隔， 默认值是1小时。
2. fs.checkpoint.size定义了edits日志文件的最大值，一旦超过这个值会导致强制执行检查点（即使没到检查点的最大时间间隔）。默认值是64MB。

> NameNode会从fs.checkpoint.dir目录读取检查点，并把它保存在dfs.name.dir目录下。如果dfs.name.dir目录下有合法的镜像文件，NameNode会启动失败。 NameNode会检查fs.checkpoint.dir目录下镜像文件的一致性，但是不会去改动它。

#### 常见的题目

1. 如果NameNode上除了最新的检查点以外，所有的其他的历史镜像和edits文件都丢失了， NameNode可以引入这个最新的检查点。以下操作可以实现这个功能：
   1. 在配置参数dfs.name.dir指定的位置建立一个空文件夹；
   2. 把检查点目录的位置赋值给配置参数fs.checkpoint.dir；
   3. 启动NameNode，并加上-importCheckpoint。

2. NameNode硬盘坏了，数据能恢复吗？如果能，能够全部恢复吗？

   (1) 能, 把Secondary NameNode的元数据拷贝给namenode  

   (2) 不能, 可能会丢失最后一次操作30min内(默认时间)或不超过64M大小数据的所有操作

3. NameNode宕机了，有Sercondary NameNode存在，所以hdfs还是只可以用的吗？

    > 不可以，Secondary NameNode只是协助进行元数据合并，不对外提供服务，所以hdfs会瘫痪。
    > 热备：你坏了，我立马顶上。

4. 有Sercondary NameNode 和NameNode我们配置时需要注意什么？

    > NameNode, Secondary NameNode不能配置在同一个节点上
    >
    > hdfs的工作目录，至少配置两个，会同时向两个目录写东西，内容完全一样。