---
title: 布隆过滤器
date: 2022-4-10 13:24:22
tags: 
- 读论文
cover_picture: https://s3.bmp.ovh/imgs/2022/04/10/af4cd8d8e5c2cc91.jpg
---

## 前言

Bigtable是管理大量结构化数据的分布式存储系统。是一个稀疏的、分布式的、持久化存储的多维度排序Map。
规模：PB级别数据，数千台机器。
性质：广泛应用、可扩展性、高可用，高性能，CAP？
不支持传统关系模型，但可以根据数据格式和部署进行动态调整（locality, 相同前缀的数据存在一起，可以做预取优化）。用户可以动态控制何时落盘，何时从磁盘读取。
其开源单机实现的版本是著名的leveldb。

## 数据模型
```go
(row:string, column:string, time:int64) -> string
```

### Rows
在单个Row下进行的读写保证是原子性的，并发更新行时方便用户编程。排序是通过对行字典序排列的。每一部分行组成tablet，是分布式存储和加载的最小单位。如果只想找几个行，就只需要几台机器的通讯就可以做到。性质相同的数据可以做连续的存储

### 列族
多个列组成的集合叫做列族。列族是访问控制的基本单位，列族层面可以控制、统计磁盘内存使用，上述的控制权限能帮助我们管理不同类型的应用：我们允许一些应用可以添加新的基本数据、一些应用可以读取基本数据并创建继承的列族、一些应用则只允许浏览数据。表可以有无限多的列，但希望只有上百个列族。同一列族下的数据往往是相同类型的，这样方便压缩。

### 时间戳

表的每一个数据都可以包含不同版本，可以通过配置列族的参数来做GC，制定保留最近的三个版本等等。

## 实现
GFS，分布式文件系统，Bigtable的表存储文件是SSTable格式的，SSTable是持久化的、排序的、不可更改的Map结构，是分块存储的，根据索引可以加载到内存。

Chubby，类似于ZK，Paxos协议，用途：
* 选主
* 存储Bigtable自引导
* 服务发现，服务器上线下线
* 存储元数据，各个表的列族信息
* 存储访问控制列表

### Tablet 分布
![tablet](/image/Tablet.png)
Chubby文件，存储root tablet 信息。
root tablet，一系列元数据表中的第一个表，存其他元数据表中tablet的位置信息，不会分裂。
其他元数据表的tablet。可以索引到用户的tablet。

### Tablet 分配
客户端直接跟Tablet服务器沟通，绕过master。
Chubby 有个文件锁，标志着该tabelt在工作，如果 Tablet 丢失了Chubby上的独占锁，就认为它已经不在工作，master会重新分配。
Master可以了解整个集群的数据分布，可以做负载均衡，Tablet分配，服务器加入和推出。Master发现tablet的文件锁被丢失了，就会自己去占领这个锁，判断是Chubby宕机还是服务器宕机。Tablets集合变化：增加删除，合并，分裂。

Master启动：Master获得Master锁，扫描chbby判断服务集群状态，与所有Tablet服务器通信获得Tablet分配的状态。（自引导）。
分裂：Tablet表太大或者访问太频繁会导致分裂。首先commit分裂操作到Metadata表中，然后通知master服务器。如果通知没成功，master会要求装载这个的子表，新的Tablet服务器加载子表的时候会查询Metadata表，发现要加载的表信息不完整，会进一步通知master。

### Tablet 服务
![tablet_server][/image/TabletRepresentation.png]
Tablet底层是LSM Tree。可以去看leveldb实现
读写操作写入WAL，数据写入内存表memtable，dump到磁盘上，
写操作要到Chubby鉴权，会有客户端缓存。提交之后会插入到memtable。
读操作要靠合并SSTable和memtable视图，合并效率很高。
合并和分割时，读写仍然可以进行。

### 压缩
memtable大小太大时触发压缩变成SSTable。
后台有合并压缩进程，定时把SSTable文件删除生成新的压缩文件。叫做主压缩。
定期回收内存资源和磁盘资源。

