---
layout: article
title: HDFS客户端数据上传与下载细节
categories: hadoop
#excerpt:
#tags: []
image:
    teaser: /teaser/osc.jpg
#    thumb:
---

### 客户端上传数据:  

![mark](http://pc06h57sq.bkt.clouddn.com/blog/181015/gaD7380Dc5.png?imageslim)  

1. 客户端上传一个文件，namenode请求上传文件, cls.avi 300M 
2. namenode响应，获取文件名，然后去元数据中查找/hadoop/hdfs/中是否已经存在相同文件名的文件，如果没有，那么告诉客户端说你可以上传这个文件.
3. 客户端拿到第一个块block01后，跟namenode说，我要上传block01, RPC请求上传第一个block(0-128M)，请求返回datanode(dn个数由客户端复本数决定)。
4. 返回(dn1, dn2, dn3) 考虑的因素：空间距离
5. 请求建立block传输通道channel
    - dn1向dn2请求建立通道 (pipe line)

    - dn2向dn3请求建立通道 (pipe line)
6. datanode应答
    - dn3应答成功, dn2应答成功, dn1应答成功
7. 开始上传
    1. 上传第一个block
    2. 数据以packet形式将数据包流式传输到pipe line的第一个datanode1，dn1存储数据并将它发送到pipe line中到达dn2。
    3. dn2通过pipe line向最后一个节点dn3传输数据并存储。
    4. 传输完成会通知客户端，block01上传成功
    5. 客户端会通知namenode说block01上传成功，此时namenode会将元数据（可以简单地理解为记录了block块存放到哪个datanode中）同步到内存中。
8. 其他块循环上述过程

#### MOST IMPORT
1. 数据传输是以packet为单位进行传输，大小默认为64K。
2. packet以chunk为单位进行校验，大小默认为512Byte。
3. 只要有一个副本上传成功即可，其余失败的副本，之后nameNode会做异步的同步。
4. 每传一个block，都需要向nameNode发出请求, 再分配客户端复本个数的datanode。


### 客户端读数据:  

![mark](http://pc06h57sq.bkt.clouddn.com/blog/181016/CD5bDe4Jjm.png?imageslim)  
