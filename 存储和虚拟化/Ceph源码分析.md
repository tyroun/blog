# Ceph源码分析
# 第一章 Ceph整体架构

## 1.1 Ceph的发展历程

![image-20210927163525811](../image/Ceph源码分析/image-20210927163525811.png)

## 1.2 Ceph的设计目标

通过集群优势来发挥高性能，通过软件的设计解决高可用性和可扩展性

## 1.3 Ceph基本架构图

<img src="../image/Ceph源码分析/image-20210927164905727.png" alt="image-20210927164905727" style="zoom:67%;" />

* 最底层是**RADOS**(reliable autonomous distributed object store)

  它是一个可靠的、自组织的、可自动修复、自我管理的分布式对象存储系统

  ceph-osd 后台服务进程  +   cpeh-mon 监控进程

* 中间层librados

  提供本地和远程接口。目前支持C/C++语言、Java、Python、Ruby和PHP语言的接口

* 面向应用提供了3种不同的存储接口

  * 块存储接口

    给虚拟机提供虚拟磁盘/ 通过内核映射给物理机提供磁盘

  * 对象存储接口

    AWS S3接口 / OpenStack Swift 接口

  * 文件系统接口

    Posix接口 / libcephfs接口 /  文件系统的meta data server (MDS) 提供meta data 访问

## 1.4 Ceph客户端接口

### 1.4.1 RBD(rados block device)

目前RBD提供了两个接口

一种是直接在用户态实现，通过QEMU Driver供KVM虚拟机使用。

另一种是在操作系统内核态实现了一个内核模块。通过该模块可以把块设备映射给物理主机，由物理主机直接访问

### 1.4.2 CephFS

CephFS通过在RADOS基础之上增加了MDS（Metadata Server）来提供文件存储。

它提供了libcephfs库和标准的POSIX文件接口。

CephFS类似于传统的NAS存储，通过NFS或者CIFS协议提供文件系统或者文件目录服务。

### 1.4.3 RadosGW

RadosGW基于librados提供了和Amazon S3接口以及OpenStack Swift接口兼容的对象存储接口

* RESTful 存储接口

* 扁平的数据组织形式

  无论是S3还是Swift，都是分3级存储。S3是Accout/Bucket/Object，Swift是Account/Container/Object

## 1.5 RADOS

* Monitor模块为整个存储集群提供全局的配置和系统信息
* 通过CRUSH算法实现对象的寻址过程
* 完成对象的读写以及其他数据功能
* 提供了数据均衡功能
* 通过Peering过程完成一个PG内存达成数据一致性的过程
* 提供数据自动恢复的功能
* 提供克隆和快照功能
* 实现了对象分层存储的功能
* 实现了数据一致性检查工具Scrub

### 1.5.1 Monitor

Monitor是一个独立部署的daemon进程。通过组成Monitor集群来保证自己的高可用。Monitor集群通过Paxos算法实现了自己数据的一致性。它提供了整个存储系统的节点信息等全局的配置信息

Cluster Map保存了系统的全局信息，主要包括：

* Monitor Map
  *  包括集群的fsid
  *  所有Monitor的地址和端口
  *  current epoch 
* OSD Map：所有OSD的列表，和OSD的状态等
* MDS Map：所有的MDS的列表和状态

### 1.5.2 对象存储

对象是数据存储的基本单元，一般默认4MB大小

![image-20210927171100622](../image/Ceph源码分析/image-20210927171100622.png)

一个对象由3部分组成

* 对象标志(ID)
* 对象数据，保存至本地文件系统的文件里
* 对象的metadata, key-value形式保存至文件的扩展属性里。 也会以leveldb这样的KV来保持

### 1.5.3 pool和PG的概念

pool是个抽象的存储池。分replicated和Erasure Code两种

pool有多个PG(placement group)组成，PG由多个对象租出，这些对象放在不同的OSD上

<img src="../image/Ceph源码分析/image-20210927171924892.png" alt="image-20210927171924892" style="zoom:67%;" />

* PG1和PG2同属于1个pool，所以是2副本
* PG1所有对象的主从副本都在OSD1和OSD2上。PG2的对象在OSD2和OSD3上
* **一个对象只能属于一个PG**
* OSD2上有两个PG的对象

### 1.5.4 对象寻址过程

#### 1 Ojbect id 到 PG的映射

静态hash映射

```C
pg_id = hash(object_id) % pg_num
```

#### 2 PG到OSD的映射

使用了CRUSH算法，其本质是一个伪随机分布算法

### 1.5.5 数据读写过程

写操作如下

<img src="../image/Ceph源码分析/image-20210927172956441.png" alt="image-20210927172956441" style="zoom:60%;" />

1. Client向PG所在的主OSD发写
2. 主OSD一边写入本地，一边向2个从OSD发写副本请求
3. 主OSD收到从OSD的Ack，并且确认自己写完，返回Ack给Client

### 1.5.6 数据均衡

数据迁移的基本单位是PG。

新加入一个OSD，会改变CRUSH Map。再引发PG到OSD列表的map改变

​																						数据迁移前PG分布

|      | OSD1 | OSD2 | OSD3 |
| ---- | ---- | ---- | ---- |
| PGa  | PGa1 | PGa2 | PGa3 |
| PGb  | PGb3 | PGb1 | PGb2 |
| PGc  | PGc2 | PGc3 | PGc1 |
| PGd  | PGd1 | PGd2 | PGd3 |

​																						数据迁移后PG分布

|      | OSD1     | OSD2     | OSD3     | OSD4 |
| ---- | -------- | -------- | -------- | ---- |
| PGa  | ~~PGa1~~ | PGa2     | PGa3     | PGa1 |
| PGb  | PGb3     | PGb1     | ~~PGb2~~ | PGb2 |
| PGbc | PGc2     | ~~PGc3~~ | PGc1     | PGc3 |
| PGd  | PGd1     | PGd2     | PGd3     |      |

### 1.5.7 Peering

某个OSD失效时，这个OSD上的主PG会发起Peering。

Peering就是同步一个PG内所有副本，通过PG日志完成。Peering完成后，改PG才可以对外进行读写服务

### 1.5.8 Recovery和Backfill

Recover在Peering过程中完成。

如果某个OSD长期失效后重新加入集群，无法根据PG日志来修复。只能执行Backfill(回填)

Backfill过程是通过逐一对比两个PG的对象列表来修复。当新加入一个OSD产生了数据迁移，也需要通过Backfill过程来完成。

### 1.5.9 纠删码(Erasure Code)

N份元数据，允许M份出错。一共存M+N份

### 1.5.10 快照和克隆

快照和克隆的区别在于快照只能读，而克隆可写

### 1.5.11 Cache Tier

RADOS以pool为单位实现分层存储。Cache Pool 和 Data Pool。通过Cache Tier来管理热点数据。

<img src="../image/Ceph源码分析/image-20211001204827477.png" alt="image-20211001204827477" style="zoom:67%;" />

### 1.5.12 Scrub

Scrub用于检查数据一致性。后台定期扫描，比较一个PG内数据和副本来检查一致性。

* 一种是只比较各个副本meta data
* 一种是deep scrub，检查数据

# 第二章 Ceph通用模块

