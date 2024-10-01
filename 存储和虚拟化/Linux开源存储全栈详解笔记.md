---
typora-copy-images-to: img typora-root-url: img
---

# Linux 开源存储全栈详解

# 第一章 Linux 开源存储

## 1.1 名字解释

- SCSI

SCSI 是小型计算机系统接口(Small Computer System Interface)的简称，SCSI 作为输入/输出接口，主要用于硬盘、光盘、磁带机、扫描仪、打印机等设备。

- FC

用于计算机设备之间数据传输，传输率达到 2G（将来会达到 4G）。光纤通道用于服务器共享存储设备的连接，存储控制器和驱动器之间的内部连接。

- DAS

DAS 是直连式存储(Direct-Attached Storage)的简称，是指将存储设备通过 SCSI 接口或光纤通道直接连接到一台计算机上。当服务器在地理上比较分散，很难通过远程进行互连时，DAS
是比较好的解决方案。但是这种式存储只能通过与之连接的主机进行访问，不能实现数据与其他主机的共享，同时，DAS 会占用服务器操作系统资源，例如 CPU 资源、IO 资源等，并且数据量越大，占用操作系统资源就越严重。

<img src="https://pic1.zhimg.com/80/v2-9c78245fa2d77d8afccd99f72466bec8_720w.jpg" alt="img" style="zoom:67%;" />

- NAS

网络接入存储(Network-Attached Storage)简称 NAS，它通过网络交换机连接存储系统和服务器，建立专门用于数据存储的私有网络，用户通过 TCP/IP 协议访问数据，采用业界标准的文件共享协议如
NFS、HTTP、CIFS 来实现基于文件级(**非块级**)的数据共享。

![img](https://pic3.zhimg.com/80/v2-643d13048c7616c6ecd65f77ac4c4e9a_720w.jpg)

- SAN

存储区域网络(Storage Area Network)简称 SAN，它是一种通过光纤交换机、光纤路由器、光纤集线器等设备将磁盘阵列、磁带等存储设备与相关服务器连接起来的高速专用子网。

SAN 由 3 个部分组成，分别是连接设备(如路由器、光纤交换机和 Hub)、接口(如 SCSI、FC)、通信协议(如 IP 和 SCSI)。这 3 个部分再加上存储设备和服务器就构成了一个 SAN 系统。

![img](https://pic4.zhimg.com/80/v2-5efda2f9626bc41a9f9dd19a3f575a3f_720w.jpg)

这里我们注意一下 SAN 架构中的连接存储设备和服务器的**专用链路（SAN 结构图中的绿色线条）**，早期是采用光纤链路的方式。目前的话已经是可以使用普通的 TCP/IP 链路（故存在 FC SAN 和 IP SAN 两种形式）。

- ISCSI

  通过 TCP/IP 传输 SCSI 命令。这样可以使用便宜的 IP SAN 来构建 SAN。

  IP SAN 存储需要专门的驱动和设备，幸运的是，一些传统的光纤适配器厂商都发布了 iSCSI HBA 设备，同时 Inter 也推出了专用的 IP 存储适配器，而 Microsoft、HP、Novell、SUN、AIX、Linux
  也具有 iSCSI Initiator 软件，并且免费供用户使用

  iSCSI 的组成

  一个简单的 iSCSI 系统大致由以下部分组成:

  iSCSI Initiator 或者 iSCSI HBA - 使用 iSCSI 的客户端

  iSCSI Target - 磁盘

  以太网交换机

  一台或者多台服务器

  CentOS 的 tgt 软件 scsi-target-utils/iscsi-initiator-utils

- ISCSI HBA/FC HBA

  HBA 是 host bus adapt, 表示存储接口和 host 接口的转换器。ISCSI 就是把 host 的 PCIe 口转成 iSCSI 协议通过 TCP/IP 送出去。**iSCSI HBA 是在 Initiator
  上的**，也就是客户端发起访问的时候转成 iSCSI 协议，在 target 处只需要把 SCSI 协议直接发给磁盘让磁盘应答就好了，所以不需要额外硬件

## 1.2 Linux 开源存储系统方案介绍

### 1.2.1 Linux 单节点存储方案

#### 1 本地文件系统

#### 2 Linux 远程存储服务

1. 块设备服务

   iSCSI 和 NVMe over Fabrics

2. 文件存储服务

   NFS/CIFS(Samba)/FTP/SSH

### 1.2.2 存储服务分类

1. 块存储服务

   LVM

    2. 文件存储服务

   POSIX 文件系统 API

    3. 对象存储服务

   put/get/delete

   没有目录层次结构

   Metadata 用于存储对象属性，有独立的 metadata 服务器

### 1.2.4 重复数据删除

Linux 下的独立开源存储方案不太多，目前用得比较多的是 OpenDedup。OpenDedup 针对 Linux 的重复数据删除文件系统被称为 SDFS

### 1.2.5 开源云计算数据存储平台

![image-20210916161140883](../image/typora-user-images/image-20210916161140883.png)

不同层次提供云计算

### 1.2.6 存储管理和软件定义存储

![image-20210916161646584](../image/typora-user-images/image-20210916161646584.png)

SNIA 认为 SDS 应该包括：

- 自动化：简化管理，降低维护存储架构的成本；

- 标准接口：提供应用编程接口，用于管理、部署和维护存储设备和存储服务；

- 虚拟数据路径：提供块、文件和对象的接口，支持应用通过这些接口写入数据；

- 扩展性：无需中断应用，也能提供可靠性和性能的无缝扩展；

- 透明性：提供存储消费者对存储使用状况及成本的监控和管理。

  data path 的虚拟化 + control path 抽象为存储服务

1. 控制平面控制平面常见的组件有以下几种。

- VMware SPBM（Storage Policy Base Management），基于存储策略的管理。
- OpenStack Cinder，用于提供块存储服务。
- EMC ViPR，其目标是实现 EMC 存储、异构存储、商用硬件本地存储资源的存储虚拟化（包括互操作性）。

2. 数据平面

- 商用硬件
    - HCI 超融合架构： VMWARE VSAN、EMC ScaleIO （超融合既分布式部署+分布式存储）
    - 非超融合架构：Dell Fluid Cache、HP StorVirtual
- 传统存储阵列 SAN+NAS
- 云和对象存储

#### SDS 开源存储软件

- OpenSDS
- libvirt Storage Management.
- OHSM

### 1.2.7 开源分布式存储和大数据解决方案

分布式存储系统一般包括三大组件：元数据服务器（也称为主控服务器）、客户端及数据服务器。

![image-20210920175117704](../image/typora-user-images/image-20210920175117704.png)

#### 1 元数据（主控）服务器

- 命名空间管理

  分布式存储系统的元数据，包括对象和文件块的索引、文件之间的关系等

- 数据服务器管理

  管理各个数据节点，故障时恢复启动节点，管理备份节点。

- 主备份容灾

  元数据和数据都需要备份。基于内存的元数据管理方式，还需要启动日志系统持久化数据。

#### 2 数据服务器

- 数据的本地存储管理

  多个小文件存在同一块里。大文件分割成小文件存放在不同节点

- 状态维护

  数据服务器将自己的状态信息报告给元数据服务器，通常这些信息会包含当前的磁盘负载、I/O 状态、CPU 负载、网络情况等

- 副本管理

#### 3 客户端

提供操作系统接口，和虚拟文件系统对接

提供用户态的用户访问接口

Restful API 接口

#### 4 常见的开源分布式存储软件

##### Hadoop

Hadoop 是由 Apache 基金会所发布的开源分布式计算平台，起源于 Google Lab 所开发的 MapReduce 和 Google 文件系统。准确来说，Hadoop
是一个软件编程框架模型，利用计算机集群处理大规模的数据集进行分布式存储和分布式计算。

Hadoop 由 4 个模块组成，即 Hadoop Common、HDFS （Hadoop Distributed File System）、Hadoop YARN 及 HadoopMapReduce。其中，主要的模块是 HDFS 和
Hadoop MapReduce。

HDFS 是一个分布式存储系统，为海量数据提供存储服务。

Hadoop MapReduce 是一个分布式计算框架，用来为海量数据提供计算服务。

##### HPCC

High Performance Computing Cluster。HPCC 是一款开源的企业级大规模并行计算平台，主要用于大数据的处理与分析。HPCC 提供了独有的编程语言、平台及架构，与 Hadoop 相比，在处理大规模数据时
HPCC 能够利用较少的代码和较少的节点达到更高的效率

##### ClusterFS

GlusterFS 是一个开源分布式存储系统，具有强大的横向扩展能力，能够灵活地结合物理、虚拟的云资源实现高可用（HighAvailability,HA）的企业级性能存储，借助 TCP/IP 或 InfiniBand RDMA
网络将物理分布的网络存储资源聚集在一起，并使用统一的全局命名空间来管理数据。同时， GlusterFS 基于可堆砌的用户空间设计，可以为各种不同的数据负载提供优质的性能。

##### Ceph

Ceph 是一款开源分布式存储系统，起源于 SageWeil 在加州大学圣克鲁兹分校的一项博士研究项目，通过统一的平台提供对象存储、块存储及文件存储服务，具有强大的伸缩性，能够为用户提供 PB 乃至 EB 级的数据存储空间。Ceph
的优点在于，它充分利用了集群中各个节点的存储能力与计算能力，在存储数据时会通过哈希算法计算出该节点的存储位置，从而使集群中负载均衡。同时，Ceph 中采用了
Crush、哈希环等方法，使它可以避免传统单点故障的问题，在大规模集群中仍然能保持稳态。目前，一些开源的云计算项目都已经开始支持 Ceph。例如，在 OpenStack 中，Ceph 的块设备存储可以对接 OpenStack 的
Cinder 后端存储、Glance 的镜像存储和虚拟机的数据存储。

##### Sheepdog

Sheepdog 是一个开源的分布式存储系统，于 2009 年由日本 NTT 实验室所创建，主要用于为虚拟机提供块设备服务。Sheepdog
采用了完全对称的结构，没有类似元数据服务器的中心节点，没有单点故障，性能可线性扩展。当集群中有新节点加入时，Sheepdog 会自动检测并将新节点加入集群中，数据自动实现负载均衡。目前 QEMU/KVM、OpenStack 及
Libvirt 等都很好地集成了对 Sheepdog 的支持。Sheepdog 总体包括集群管理和存储管理两大部分，运行后将启动两种类型的进程：sheep 与 dog，其中，sheep 进程作为守护进程兼备节点路由及对象存储功能；dog
进程作为管理进程可管理整个集群。在 Sheepdog 对象存储系统中，getway 负责从 QEMU 的块设备驱动上接收 I/O 请求，并通过哈希算法计算出目标节点，将 I/O 转发到相应的节点上。

# 第三章 Linux 存储栈

## 3.1 Linux 存储系统概述

![Linux-storage-stack-diagram_v4.10](../image/img/Linux-storage-stack-diagram_v4.10.svg)

POSIX 文件接口 -> VFS -> Page Cache -> BIO -> (LVM/bdcache) -> 块层 -> 块设备 -> 不同协议 -> 物理设备

## 3.2 系统调用

pmap 进程号 或者 /proc/<pid>/maps 查看当前进程的内存分布

```shell
pmap 4855
4855:   -bash
0000000000400000    976K r-x-- bash
00000000006f3000      4K r---- bash
00000000006f4000     36K rw--- bash
00000000006fd000     24K rw---   [ anon ]
00000000015c6000   5600K rw---   [ anon ]
00007ff7c56f0000     44K r-x-- libnss_files-2.23.so
00007ff7c56fb000   2044K ----- libnss_files-2.23.so
00007ff7c58fa000      4K r---- libnss_files-2.23.so
00007ff7c58fb000      4K rw--- libnss_files-2.23.so
00007ff7c58fc000     24K rw---   [ anon ]
00007ff7c5902000     44K r-x-- libnss_nis-2.23.so
00007ff7c590d000   2044K ----- libnss_nis-2.23.so
00007ff7c5b0c000      4K r---- libnss_nis-2.23.so
00007ff7c5b0d000      4K rw--- libnss_nis-2.23.so
00007ff7c5b0e000     88K r-x-- libnsl-2.23.so
00007ff7c5b24000   2044K ----- libnsl-2.23.so
00007ff7c5d23000      4K r---- libnsl-2.23.so
00007ff7c5d24000      4K rw--- libnsl-2.23.so
00007ff7c5d25000      8K rw---   [ anon ]
00007ff7c5d27000     32K r-x-- libnss_compat-2.23.so
00007ff7c5d2f000   2044K ----- libnss_compat-2.23.so
00007ff7c5f2e000      4K r---- libnss_compat-2.23.so
00007ff7c5f2f000      4K rw--- libnss_compat-2.23.so
00007ff7c5f30000   4652K r---- locale-archive
00007ff7c63bb000   1792K r-x-- libc-2.23.so
00007ff7c657b000   2048K ----- libc-2.23.so
00007ff7c677b000     16K r---- libc-2.23.so
00007ff7c677f000      8K rw--- libc-2.23.so
00007ff7c6781000     16K rw---   [ anon ]
00007ff7c6785000     12K r-x-- libdl-2.23.so
00007ff7c6788000   2044K ----- libdl-2.23.so
00007ff7c6987000      4K r---- libdl-2.23.so
00007ff7c6988000      4K rw--- libdl-2.23.so
00007ff7c6989000    148K r-x-- libtinfo.so.5.9
00007ff7c69ae000   2044K ----- libtinfo.so.5.9
00007ff7c6bad000     16K r---- libtinfo.so.5.9
00007ff7c6bb1000      4K rw--- libtinfo.so.5.9
00007ff7c6bb2000    152K r-x-- ld-2.23.so
00007ff7c6dac000     16K rw---   [ anon ]
00007ff7c6dd0000     28K r--s- gconv-modules.cache
00007ff7c6dd7000      4K r---- ld-2.23.so
00007ff7c6dd8000      4K rw--- ld-2.23.so
00007ff7c6dd9000      4K rw---   [ anon ]
00007ffc23556000    132K rw---   [ stack ]
00007ffc23595000     12K r----   [ anon ]
00007ffc23598000      4K r-x--   [ anon ]
ffffffffff600000      4K r-x--   [ anon ]
 total            28256K
```

如果映射的文件名为［anon］，则表示匿名的内存映射，指动态生成的内容所占用的内存

注意一下最后一个［anon］区域，它的地址在内核地址空间，这是 VDSO（Virtual Dynamic Shared Object）技术的应用，相当于将一个内核动态库共享给应用进程。

> VDSO - vsyscall 是老的系统调用加速方法，vdso 是新的。vsyscall 固定一段内核地址，系统调用时用这段地址做函数调用。vdso 不用固定地址
>
> https://vvl.me/2019/06/linux-syscall-and-vsyscall-vdso-in-x86/

一般属性为“r-x”（只读并可执行的）的是程序的代码段，而具有可写属性的可能是数据段、未初始化的全局变量段、堆、栈等

## 3.3 文件系统

### 3.3.1 文件系统概述

虚拟文件系统主要有如下 4 个对象类型

- 超级块 super block
- 索引节点 inode
- 目录项 Dentry
- 文件

### 3.3.2 Btrfs

#### 1 B-Tree

Btrfs 所有元数据使用 B-Tree 管理，ext4 才开始对目录索引超过一定数量的使用 B-Tree 管理，否则都是线性查找

B-Tree 和 2 叉树比，减少了树的高度，也就降低了磁盘 IO 的访问次数

#### 2 基于 extent 的文件存储

ext2 的时候，每个 4K 块都有一个指针，大文件需要很多指针。ext4、Btrfs 都支持 extent 功能，用 extent 取代 block 来管理磁盘，能有效地减少元数据开销。extent 由一些连续的 block
组成，由起始的 block 加上长度进行定义。

#### 3 动态 inode 分配

在 ext2/3/4 系列的文件系统中，索引节点的数量都是固定的，如果存储很多小文件的话，就有可能造成索引节点已经用完。不过它也有一个好处，就是磁盘一旦损坏，恢复起来要相对简单，因为数据在磁盘上的布局是相对固定的。

为了解决磁盘剩余空间无法被使用的问题，需要对索引节点进行动态分配：每一个索引节点作为 B-Tree 中的一个节点，用户可以无限制地任意插入新的索引节点，索引节点的物理存储位置是动态分配的，所以 Btrfs 没有对文件个数进行限制。

#### 4 固态硬盘优化

#### 5 元数据的 CRC 校验

Btrfs 采用 crc32 算法计算校验和

ext4 也有校验

#### 6 写时复制

对 Btrfs 来说，所谓的写时复制，是指每次将数据写入磁盘时，先将更新的数据写入一个新的 block，当新数据写入成功之后，再更新相关的数据结构指向新 block。写时复制只能保证单一数据更新的原子性

#### 7 子分区 subvolume

subvolume 可以作为根目录，挂载到任意挂载点。假如管理员只希望用户访问文件系统的一部分，如希望用户只能访问/var/test/下面的内容，而不能访问/var/目录下的其他内容，那么便可以将/var/test 做成一个
subvolume。/var/test 的这个 subvolume 便是一个完整的文件系统，可以用 mount 命令挂载。如果挂载到/test 目录下，给用户访问/test 的权限，那么用户便只能访问/var/test 下面的内容。

#### 8 软件 RAID

Btrfs 很好地支持了软件磁盘阵列，包括 RAID0、RAID1 和 RAID10。

#### 9 压缩

Btrfs 已经能够支持 zlib、LZO 与 zstd 等方式进行压缩。

## 3.4 Page Cache

应用层对文件的访问一般有两种方法：一是通过系统调用 mmap()创建直接访问的虚拟地址空间，程序执行；二是利用系统调用 read()/write())进行寻址访问。

一个文件通过 mmap()
映射到虚拟内存空间后，对这个内存区域进行第一次访问时，页表还没有建立，必然会出现一个内存访问的缺页错误。内核在处理这个缺页错误时，通过页面预读函数分配内存页面，然后将对应的文件块读入。当程序再次访问这个文件块的时候，由于它已经在内存缓存中了，因此就不需要再次访问文件了。

通过系统调用 read()来访问文件时，最终也是通过页面预读函数分配 PageCache 的。而对文件缓存的写操作，使用写时复制机制，等到要同步或清理缓存时再把文件同步回去，这种情况一般会有延迟效应。

在早期版本的内核中，Page Cache 和 Buffer Cache 是两个独立的缓存，前者缓存页，后者缓存块，从 2.4.10 版本内核开始，Buffer Cache 不再是一个独立的缓存了，它被包含在 Page Cache 中，通过
PageCache 来实现。对于 4KB 大小的 page 来说，根据不同的块大小，它可以包含 1 ～ 8 个缓冲区。

```shell
free
              total        used        free      shared  buff/cache   available
Mem:       16259400     6617064     3143312     1028100     6499024     5175976
Swap:             0           0           0
```

“buffers”表示块设备所占用的缓存页，包括直接读/写块设备，以及文件系统元数据，如 SuperBlock 所使用的缓存页；“cached”表示普通文件所占用的缓存页。

## 3.6 块层

### 3.6.1 bio 与 request

**bio 描述了磁盘里要真实操作的位置与 Page Cache 中页的映射关系**

每个 bio 对应磁盘里面一块连续的位置，对应 Page Cache 的多页或一页，所以磁盘里面会有一个 structbio_vec\*bi_io_vec 的表

```c
struct bio_vec {
	struct page *bv_page;
	unsigned int bv_len;
	unsigned int bv_offst;
}
```

<img src="../image/typora-user-images/image-20210920225413594.png" alt="image-20210920225413594" style="zoom:50%;" />

bio 在块层会被转化为 request，多个连续的 bio 可以合并到一个 request 中，生成的 request 会继续进行合并、排序，并最终调用块设备驱动的接口将 request 从块层的 request
队列（request_queue）中移到驱动层进行处理，以完成 I/O 请求在通用块层的整个处理流程。

**request 用来描述单次 I/O 请求**

## 3.7 LVM

在 2.6 内核，基于 device mapper 机制实现了第二个版本 LVM2。

<img src="../image/typora-user-images/image-20210920225836095.png" alt="LVM分层" style="zoom:30%;" />

LVM 分层

<img src="../image/typora-user-images/image-20210920230513311.png" alt="image-20210920230513311" style="zoom:50%;" />

​ device mapper 架构

整个 device mapper 机制由两部分组成：内核空间的 devicemapper 驱动，用户空间的 device mapper 库

device mapper 在内核中被注册为一个块设备驱动，它包含 3 个重要的对象概念：

- Mapped device 是一个逻辑抽象，可以理解为内核向外提供的逻辑设备，它通过 Mapping table 描述的映射关系和 Target device 建立映射。
- Target device 表示的是 Mapped device 所映射的物理空间段，对 Mapped device 所表示的逻辑设备来说，就是该逻辑设备映射到的一个物理设备。
- Mapping table 里有 mapped device 逻辑上的起始地址、范围和表示在 Target device 上的地址偏移量及 Target 类型等信息

device mapper 中的逻辑设备 Mapped device 不但可以映射一个或多个物理设备 Target device，还可以映射另一个 Mapped device

## 3.8 bcache

bcache 是 Linux 内核的块层缓存，它使用固态硬盘作为硬盘驱动器的缓存

bcache 从 3.10 版本开始被集成进内核，支持 3 种缓存策略，分别是写回（writeback）、写透（writethrough）、writearoud，默认使用 writethrough，缓存策略可被动态修改

bucket 是 bcache 最关键的结构，缓存设备按照一定的大小划分成很多 bucket,bucket 的默认大小是 512KB，不过最好将其与缓存设备固态硬盘的擦除大小设置一致，如图 3-14 所示的每个小的方框都代表一个
bucket，缓存数据与元数据都是按 bucket 来管理。

bcache 采用写时复制的方式以 bucket 为单位进行空间的分配，对已经在固态硬盘中的数据与元数据进行覆盖写操作时，是写到新的 bucket 空间中的，这样无效的旧数据就会在其所在的 bucket
内形成“空洞”。因此需要一个异步的垃圾回收（GC）线程对这些数据进行标记与清理，并将含有较多无效数据的多个 bucket 压缩成一个 bucket。

Linux 开源社区有多个通用的块级缓存解决方案，其中包括 bcache、dm-cache、flashcache、enhanceIO 等。其中，dm-cache 与 bcache 分别于 3.9、3.10
版本合并进内核，flashcache 虽然没有引入内核，但在某些生产环境中也有使用。enhanceIO 衍生于 flashcache 项目。

作为 Linux 内核的一部分，dm-cache 也是基于 device mapper 实现的。dm-cache 可以采用一个或多个快速设备作为后端慢速存储设备的缓存。dm-cache 采用 3 个物理存储设备混合成 1
个逻辑卷的形式。其中，原始设备通常是硬盘或 SAN，提供主要的慢速存储；缓存设备通常是指固态硬盘；元数据设备记录了原始数据块在缓存中的位置、脏标志，以及执行缓存策略所需的内部数据。

## 3.9 DRBD

分布式块设备复制（Distributed Relicated Block Device,DRBD）用于通过网络在服务器之间对块设备（硬盘、分区、逻辑卷等）进行镜像，以解决磁盘单点故障的问题。可以将 DRBD
视作一种网络磁盘阵列，允许用户在远程服务器上建立一个本地块设备的实时镜像。

# 第八章 OpenStack 存储

OpenStack 比较重要的组件有计算、对象存储、认证、用户界面、块存储、网络和镜像服务

1. 计算

   计算的项目代号是 Nova，它根据需求提供虚拟机服务，如创建虚拟机或对虚拟机做热迁移等。从概念上看，它对应于 AWS 的 EC2 服务

2. 对象存储

   对象存储的项目代号是 Swift，它允许存储或检索对象，也可以认为它允许存储或检索文件，它能以低成本的方式通过 RESTful API 管理大量无结构数据。它对应于 AWS 的 S3 服务

3. 认证

   认证的项目代号是 Keystone，为所有 OpenStack 提供身份验证和授权服务

4. 用户界面

   用户界面的项目代号是 Horizon

5. 块存储

   块存储的项目代号是 Cinder，提供块存储服务。Cinder 最早是由 Nova 中的 nova-volume 服务演化而来的。Cinder 对应于 AWS EBS 块存储服务

6. 网络

   网络的项目代号是 Neutron，用于提供网络连接服务，允许用户创建自己的虚拟网络并连接各种网络设备接口。

7. 镜像服务

   镜像服务的项目代号是 Glance，它是 OpenStack 的镜像服务组件

## 8.1 Swift

### 8.1.1 Swift 体系结构

作为对象存储的一种，Swift 比较适合存放静态数据。

如下图所示，Swift = 访问层(Access Tier) + 存储层(Storage Node)

<img src="../image/img/Swift体系.png" alt="Swift体系结构" style="zoom:50%;" />

Access Tier = Proxy Node + Authentication

在 Proxy Node 上运行的 Proxy Server 用来负责处理用户的 RESTful 请求，在接收到用户请求时，需要对用户的身份进行认证，此时用户所提供的身份资料会被转发给认证服务进行处理。Proxy Server 可以使用
Memcached 进行数据和对象的缓存，来减少数据库读取的次数，提高用户的访问速度。

#### 1 存储层分层

存储层在物理上又分为如下一些层次

- Region: 地理上隔绝的区域
- Zone：在每个 Region 的内部又划分了不同的 Zone 来实现硬件上的隔绝。一个 Zone 可以是一个硬盘、一台主机、一个机柜或一个交换机，我们可以简单理解为一个 Zone 代表了一组独立的存储节点
- Storage Node：存储对象数据的物理节点，基于通用标准的硬件设备提供的对象存储服务
- Device：可以简单理解为磁盘
- Partition：这里的 Partition 仅仅是指在 Device 上的文件系统中的目录

#### 2 Storage Node 分层

每个 Storage Node 上存储的对象在逻辑上又由 3 个层次组成：Account、Container 及 Object。

<img src="../image/img/Swift对象组织结构.png" alt="Swift对象组织结构" style="zoom:=75%;" />

Account 在对象的存储过程中实现顶层的隔离，代表的并不是个人账户，而是租户，一个 Account 可以被多个个人账户共同使用；Container 代表了一组对象的封装，类似文件夹或目录，但是 Container 不能嵌套。

#### 3 Storage Node 的服务

Storage Node 上运行了 3 种服务。

- Account Server：提供 Account 相关服务，包括 Container 列表及 Account 的元数据等。Account 的信息被存储在一个 SQLite 数据库中。
- Container Server：提供 Container 相关服务，包括 Object 的列表及 Container 的元数据等。与 Account 一样，Container 的信息也被存储在一个 SQLite 数据库中。
- Object Server：提供对象的存取和元数据服务，每个对象的内容会以二进制文件的形式存储在文件系统中，元数据会作为文件的扩展属性来存储，也就是说，存储对象的物理节点上，本地文件系统必须支持文件的扩展属性，有些文件系统（如
  ext3）的文件扩展属性默认是关掉的。

#### 4 如何管理副本

为了保证数据在某个存储硬件损坏的情况下也不会丢失，Swift 为每个对象建立了一定数量的副本（默认为 3 个），并且每个副本存放在不同的 Zone 中，这样即使某个 Zone 发生故障，Swift 仍然可以通过其他 Zone 继续提供服务。

Swift 管理副本的粒度是 Partition，并非是单个对象

#### 5 数据一致性

Swift 通过以下 3 种服务来解决数据一致性的问题

- Auditor：通过持续扫描磁盘来检查 Account、Container 和 Object 的完整性，如果发现数据有损坏的情况，Auditor 就会对文件进行隔离，然后通过 Replicator
  从其他节点上获取对应的副本用以恢复本地数据。
- Updater：在创建一个 Container 的时候需要对包含该 Container 的 Account 的信息进行更新，使得该 Account 数据库里面的 Container 列表包含这个新创建的
  Container。同样在创建一个新的 Object 的时候，需要对包含该 Object 的 Container 的信息进行更新。对于那些没有成功更新的操作，Swift 会使用 Updater 服务继续处理这些失败的更新操作。
- Replicator：负责检测各个节点上的数据及其副本是否一致。当发现不一致时会将过时的副本更新为最新版本，并且负责将标记为删除的数据真正从物理介质上删除。

#### 6 存储对象和物理位置的映射

Swift 引入了环的概念

环通过 Zone、Device、Partition 和 Replica 的概念来维护映射信息。每个 Partition 的位置由环来维护，并存储在映射中。环需要在 Swift
部署时使用“swift-ring-builder”工具手动进行构建，之后每次增减存储节点时，都需要重新平衡环文件中的项目，以保证系统因此而发生迁移的文件数量最少。

### 8.1.2 环

#### 1 一致性 Hash 算法

- 计算每个对象名称的哈希值并将它们均匀地分布到一个虚拟空间上，一般用 2^32^标识该虚拟空间。
- 假设有 2^m^个存储节点，那么将虚拟空间均匀分成 2^m^份，每一份长度为 2^(32-m)^。
- 假设一个对象名称哈希之后的结果是 n，那么该对象对应的存储节点即为 n/2^(32-m)^，转换为二进制位移操作，就是将哈希之后的结果向右位移(32-m)位

#### 2 环数据结构

环包括以下 3 种重要的数据结构

- 设备表：Swift 将所有 Device 进行编号，设备表中的每一项都对应一个 Device，其中记录了该 Device 的具体位置信息，包括 Device ID，所在的 Region、Zone、IP 地址及端口号，以及用户为该
  Device 定义的权重等。

- 当 Device 的容量大小不一致时，可以通过权重保证 Partition 均匀分布，容量较大的 Device 拥有更大的权重，也容纳更多的 Partition。例如，一个 1TB 大小的 Device 有 100 的权重而一个 2TB
  大小的 Device 将有 200 的权重。

- 设备查询表（Device Lookup Table）：存储 Partition 的各个副本（默认为 3 个）与具体 Device 的映射信息。设备查询表中的每 1 列对应 1 个 Partition，每 1 行对应 Partition
  的 1 个副本，每个表格中的信息为设备表中 Device 的编号，根据这个编号，可以在设备表中检索到该 Device 的具体信息（Device ID、IP 地址及端口号等信息）。

- Partition 移位值（Partition Shift Value）：表示在哈希之后将 Object 名字进行二进制位移的位数。

  <img src="../image/img/环数据结构.png" alt="环数据结构" style="zoom:55%;" />

每个 Partition 有 3 个副本，每个单元格数字表示节点名称。

对象到 Partition 这层映射是通过哈希算法及二进制位移操作的，Partition 到存储节点的映射是通过设备查询表完成的

#### 3 构建环

Swift 使用 swift-ring-builder 工具构建一个环，所谓构建环就是构建设备查询表的过程。构建过程分为 3 个步骤

- 创建环

```shell
swift-ring-builder <builder_file> create <part_power> <replicase> <min_part_hours>
```

- 添加设备

  ```shell
  swift-ring-builder <builder_file> add [r<region>] z<zone>-<ip>:<port>/<device_name>_<meta> <weight
  ```

- 分配 Partition

  ```shell
  swift-ring-builder <buidler_file> rebalance
  ```

### 8.1.6 数据一致性

#### 1.NWR 策略

Swift 保证数据一致性的理论依据是 NWR 策略（又称为 Quorum 仲裁协议），其中，N 为数据的副本总数，W 为更新一个数据对象时需要确保成功更新的份数，R 为读取一个数据时需要读取的副本个数。

如果 W+ R> N，那么就可以保证某个数据不能被两个不同的事务同时读/写。否则，如果有两个事务同时对同一数据进行读/写，那么在 W+ R> N 的情况下，至少会有一个副本发生读/写冲突。如果 W>
N/2，那么可以保证两个事务不能并发写同一个数据，否则，至少会有一个副本发生写冲突。

Swift 默认采用了 N=3、W=2、R=2 的设置，表示一个对象默认有 3 个副本，至少需要更新 2 个副本才算写成功，至少读 2 个副本才算读成功。如果 R=1，则可能会读取到旧版本的数据。

### 2.Auditor、Updater 与 Replicator

Auditor 负责数据的审计，检查完整性

Replicator 负责数据迁移

Updater 对那些因为负荷不足而导致失败的 Account 或 Container 进行更新操作

## 8.2 Cinder

### 8.2.1 Cinder 体系结构

Cinder 则是在虚拟机与具体存储设备之间引入了一层“逻辑存储卷”的抽象。Cinder 提供的 RESTful API 则主要是针对逻辑存储卷的管理

<img src="../image/img/cinder架构.png" alt="cinder架构" style="zoom:55%;" />

- cinder-api 是进入 Cinder 的 HTTP 接口
- cinder-volume 是运行在存储节点上管理具体存储设备的存储空间，每个存储节点上都会运行一个 cinder-volume 服务，多个这样的节点便构成了一个存储资源池。
- cinder-scheduler 会根据预定的策略（如不同的调度算法）选择合适的 cinder-volume 节点来处理用户的请求

Cinder 只提供了对逻辑卷的抽象，但不提供存储服务。比如备份，快照这些。

Cinder 默认使用 LVM 作为后端存储，也可以使用其他存储系统
