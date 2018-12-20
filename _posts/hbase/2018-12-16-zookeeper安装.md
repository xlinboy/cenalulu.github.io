---
layout: article
title:  "zookeeper的安装"
categories: hbase
date:   2018-12-18 08:44:13
tags: hbase base grammer
---

### 为分布式应用提供协调服务

```
角色：主要是leader、follower
	观察者模式
		发生信息变化，就会通知到slave

单机模式：
	安装
	修改conf/zoo.cfg下的datadir即可，表示的是-存放数据的路径。
		datadir=/opt/programs/zookeeper-3.4.12/data/zkData`
	启动
		bin/zkServer.sh start
	出现进程名为QuorumPeerMain即启动成功
		bin/zkServer.sh 
		status，此时查看状态为standalone
	客户端
		bin/zkCli.sh
		客户端可以做一些类似linux文件系统的操作，
		如：create /test 
		"test"，创建的时候必须给每个节点一个内容
			查看节点信息，get /test即可
			删除节点，rmr /test即可
	- 停止服务：bin/zkServer.sh stop
```

集群模式：

	 1.配置文件需要自己增加集群选项
		表明集群有几台机器，并且指定和leader通信的端口号和参与投票的端口号
		server.1=hadoop1:2888:3888
		server.2=hadoop2:2888:3888
		server.3=hadoop3:2888:3888
	2.在每个节点上创建文件，表明本节点的编号，要和配置文件里面对应
		在data/zkData下创建文件myid，里面表明自己是哪个编号即可
		touch myid
	3.传输到别的节点上，并修改myid为对应的编号
	启动:
		只能每个节点都启动脚本
		bin/zkServer.sh start
		
		每个节点启动了之后，可以查看各节点的角色，
		bin/zkServer.sh status


​	
​	
​	
​	
​	
​	
