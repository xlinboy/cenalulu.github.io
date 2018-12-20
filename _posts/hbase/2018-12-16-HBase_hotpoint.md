---
layout: article
title:  "Hbase知识点"
categories: hbase
date:   2018-12-18 08:44:13
tags: hbase base grammer
description: 'Hbase的一些题目'
---

#### 1. hbase:meta放在哪里，以什么形式存在？  

> hbase:meta 以表的形式存放在hbase中的regionserver 上。  
hbase:meta 保存所有的regions列表。 zookeeper中保存所有的位置信息。


#### 2. client 读写细节
> client 通过查找 hbase:meta 表，找到指定提供服务的regionservers。并且获取hbase:meta的元数据。在定位所需的regions后，client联系 指定区域的regionservers为regions提供服务，而不是通过hmaster，然后开始执行读写。  
> 
> hbase:meta 被缓存在内存中。假如该region分裂或者regionserver宕了，client 重新查询meta表，以确定用户区域新位置。

#### 3. client无论读写都要找zookeeper为什么？
- zookeeper上的ZNode，有一个叫meta-region-server，保存的就是meta表在哪个regionserver
- meta表记录了我们要找的具体table在哪个regionserver

#### 4. masterb也要去找zookeeper为什么？
> master需要知道regionserver在台主机上，zookeeper保存了当前的regionserver

#### 5. hbase数据存储
1. 所有的数据都是存储在hdfs上，主要包括hfile, storefile
2. Hfile: Hbase中keyvalue数据的存储格式，HFile是hadoop二进制格式文件。
3. StoreFile: 对HFile做轻量级封装。 可以说是StoreFile就是HFile
4. HLogFile，HBase中WAL的存储，物理上的squence file

#### memstore和storefile
- client写入时，会先定入memstore, memstore写满时就会flush成一个storefile
- storefile的数量增长到一定阈值时，会触发compact合并操作，合并成一个storefile，此时会有版本合并和数据删除（此时才是真正的删除，原先删除操作只是打标签）
- 当storefile 容量增长到一定阈值，会触发spilt操作，将当前region分成2个region, 此时region会下线，新生成的hmaster会被hmaster分配到对应region server上。


**HBase只是增加数据，更新和删除操作都是在compact阶段做，所以，用户写操作只需要进入内存即可立即返回，保证了IO高性能**

#### HLog文件结构 
WAL，日志预处理，用于灾难恢复。老版本中每个regionserver维护一个Hlog，而不是每个region
- 不同的region日志会混在一起，这样做的目的是不断追加单个文件相对于写多个文件来说，可以减少磁盘寻址次数，可以提高table的写性能。
- 缺点是，如果一台regionserver下线，为了恢复其上的region，需要将该regionserver上的log进行拆分，分发到其他的regionserver进行恢复。

新版本可以设置每个region维护一个log, 优缺点反之 。





