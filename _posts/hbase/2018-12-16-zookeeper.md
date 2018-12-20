---
layout: article
title:  "Zookeeper介绍及选举机制"
categories: hbase
date:   2018-12-18 08:44:13
tags: hbase base grammer
---

# Zookeeper 概念简介

Zookeeper是一个分布式协调服务；就是为用户的分布式应用程序提供协调服务。
1. 为分布式程序提供服务
2. 本身是一个分布式程序，只要半数以上存活,zk正常提供服务
3. zookeeper 涵盖：主从协调、统一名称服务...
4. zookeeper 底层只提供两个功能：
    - 管理（存储，读取）用户程序提交的数据
    - 并为用户程序提供数据节点监听服务
    

zookeeper默认只有leader和follower，没有observer。  

集群通过选举过程来选举来选定leader的机器， leader服务器为客户端提供读和写服务。  

Follower和Observer都能提供服务，不能提供写服务。Observer不参与Leader选举，也不参与写操作（过半写成功）策略。

# zookeeper工作机制
- 基于观察者模式设计的分布式服务管理框架。  
- 存储和管理大家都关心的数据（可自定义），然后接受观察者注册
- 一旦这些数据状态发生变化，zookeeper就将负责通知已经在zookeeper上注册的那些观察者(observer)做出相应的反应


**zookeeper = 文件系统 + 通知机制**

# zookeeper 结构和特性

## zookeeper 特性
1. 主从，一个leader, 多个follower组成的集群
2. 全局数据一致：每个server保存一份相同的数据副本，client无论连接到哪个server，数据都是一致的
3. leader：分布式读写，更新请求转发
4. 同一个client请求，FIFO
5. 数据更新原子性。一次数据要么成功，要么失败。
6. 实时性，一定时间范围内,client能读最新数据。

## zookeeper 数据结构
1. 树型结构
2. 每个节点在zookeeper中叫Zonde，并且有唯一标识
3. 每个Zonde默认能够存储1MB数据，包含数据和子节点
4. 客户端应用可以在节点上设置监听器

## 节点类型
1. Zonde有两种类型
    - 短暂 ephemeral, 断开链接即删除
    - 持久 persistent, 断开链接不删除
2. Zonde有四种形式的目录节点（默认是persistent）
    - PERSISTENT
    - PERSISTENT_SEQUENTIAL（持久序列/test0000000019 ）
    - EPHEMERAL
    - EPHEMERAL_SEQUENTIAL
3. 创建znode时设置顺序标识，znode名称后会附加一个值，顺序号是一个单调递增的计数器，由父节点维护
4. 分布式系统中，顺序号可以被用于所有的事件，进行全局排序，这样客户端可以通过顺序号推断事件的顺序。

### 状态信息
每个ZNode除了存储数据内容之外，还存储了ZNode本身的一些状态信息。用 get 命令可以同时获得某个ZNode的内容和状态信息。如下：
```
cZxid = 0xb00000002	事务id
ctime = Thu Nov 01 02:32:12 CST 2018	创建时间
mZxid = 0xb0000000e	最后更新的事物id
mtime = Thu Nov 01 02:39:36 CST 2018	最后修改的时间
pZxid = 0xb0000000f	最后更新的子节点事务id
cversion = 1	子节点修改次数
dataVersion = 1	数据变化号
aclVersion = 0	访问控制列表的变化号
ephemeralOwner = 0x0	临时节点的拥有者sessionid，若非临时节点，则为0
dataLength = 5	数据长度
numChildren = 1	子节点数量
```
### Watcher
> Watcher（事件监听器），是ZooKeeper中一个很重要的特性。ZooKeeper允许用户在指定节点上注册一些Watcher，并且在一些特定事件触发的时候，ZooKeeper服务端会将事件通知到感兴趣的客户端上去。该机制是ZooKeeper实现分布式协调服务的重要特性。

#### 监听器原理： * 
![mark](http://pc06h57sq.bkt.clouddn.com/blog/181113/kDb2J2mA4i.png?imageslim)  

1. main()方法，启动client
2. 在main线程中，创建client,这时会创建两个线程，一个负责网络连接通信(connect), 一个负责监听(listener)。
3. 通过connect线程将注册的监听时间发送给zookeeper
4. 在zookeeper的注册监听器列表中将注册的监听时间添加到列表
5. zookeeper监听到有数据或路径变化，就会将消息发送给listener线程
6. listener线程内部调用process()方法。

### 写数据流程
![mark](http://pc06h57sq.bkt.clouddn.com/blog/181113/3KdEaimBf6.png?imageslim)  

1. client 连接到集群中的某一个节点
2. client 向server1发送写请求
3. 若server1 不是leader，server转发给leader
    >  leader会将写请求广播给各个server，leader会认识数据写成功了，并通知给server1
4. 若半数以上的server都写成功了，leader会认为写操作成功，并通知server1
5. server1通知client，数据写成功了。

## zookeeper 的选举机制 *
1.  半数机制： 投票数达半数以上为leader
2.  自私制+大数制

**假设有5台机器**

1. node1先启动，投票给自己，5台机器只有1票，作废
2. node2启动，node1投票给自己没有用，投票给node2，node2也投给自己，node2有两票
3. node3启动，node1投给myid最大的节点，node3，node2也投node3，node3也投自己，node3有3票，node3为leader
4. node4启动， node5启动
   


