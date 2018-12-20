### 技术选型
读取 HBase 相对来说方案比较确定，基本根据需求设计 RowKey，然后根据 HBase 提供的丰富 API（get，scan 等）来读取数据，满足性能要求即可。

写入Hbase的方法大致以下几种：
1.  Java(python等) 调用 HBase 原生 API，HTable.add(List(Put))。
2.   MapReduce 作业，使用 TableOutputFormat 作为输出。
3.   Bulk Load，先将数据按照 HBase 的内部数据格式生成持久化的 HFile 文件，然后复制到合适的位置并通知 RegionServer ，即完成海量数据的入库。其中生成 Hfile 这一步可以选择 MapReduce 或 Spark。

Spark + Bulk Load 写入 HBase优势：
1.  BulkLoad 不会写 WAL，也不会产生 flush 以及 split。
2.  如果我们大量调用 PUT 接口插入数据，可能会导致大量的 GC 操作。除了影响性能之外，严重时甚至可能会对 HBase 节点的稳定性造成影响，采用 BulkLoad 无此顾虑。
3.  过程中没有大量的接口调用消耗性能。
4.  可以利用 Spark 强大的计算能力。

架构如下:
<br/>

<img src="https://xlactive-1258062314.cos.ap-chengdu.myqcloud.com/bulk_load.PNG" style="width:80%; height:60%;" />   

### 表设计

**重点Rowkey设计**
check 表（原表字段有 18 个，为方便描述，本文截选 5 个字段示意）

<img src="https://xlactive-1258062314.cos.ap-chengdu.myqcloud.com/check.jpg"  style="width:80%; height:60%;" />





