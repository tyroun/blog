# 路线图

![f0f2e0543f9f1ee06222ff664ba9ea92.png](https://img-blog.csdnimg.cn/img_convert/f0f2e0543f9f1ee06222ff664ba9ea92.png)

# Trap-and-emulate

QEMU 虚拟机里面设备驱动需要访问该虚拟设备的寄存器时，这条访问指令会被 trap 到 QEMU，由 QEMU 来进行处理

QEMU 内核态 -> QEMU 用户态 -> QEMU 内核态

# Virtio

virtio需要在guest里面运行virtio driver。这个驱动和QEMU虚拟的设备进行数据交互，因为这个驱动知道自己操作的是虚拟设备，所以在真正的 I/O 路径上规避了大量可能导致陷入&陷出的 mmio/pio 操作，从而提高了性能。

本质上是一套基于共享内存 + 环形队列的通信机制。核心数据结构（split virtqueue）包括：两个 ringbuffer (avail ring, used ring) 和一个 descriptor table。

通过eventfd机制，和宿主机上的virtio backend通讯

<img src="https://img-blog.csdnimg.cn/img_convert/2273f9e312d6446d12d9a5eecd1bbffa.png" alt="2273f9e312d6446d12d9a5eecd1bbffa.png" style="zoom:0%;" />

# Vhost

数据收发部分直接offload到宿主机的一个内核线程。这样 virtio 通信机制从原本的 QEMU 用户态 I/O 线程和虚机驱动（QEMU 用户态 vcpu 线程）通信变成了 vhost 内核 I/O 线程和虚机驱动（QEMU 用户态 vcpu 线程）通信

![861bf60db99a50e87028730e7acf1770.png](https://img-blog.csdnimg.cn/img_convert/861bf60db99a50e87028730e7acf1770.png)

# VFIO

VFIO通过硬件IOMMU和EPT的支持，直接把硬件设备透传进了虚拟机。guest的驱动可以通过映射后的地址安全地访问设备。

Intel CPU在处理器级别加入了对内存虚拟化的支持。即扩展页表EPT。guest的物理地址到host物理地址的转换。

[EPT详细说明](https://www.cnblogs.com/ck1020/p/6043054.html)

![VFIO图](https://img-blog.csdnimg.cn/img_convert/e1b483b9d487ea8081f1aac5e3ca9ca5.png "VFIO 架构")

# Vhost-user

VFIO因为有特定的IO地址映射，无法做热迁移。所以vhost-user 提出了一种新的方式，即将 virtio 设备的数据面 offload 到另一个专用进程来处理

![6261c6e131b50419f5625475428742f5.png](https://img-blog.csdnimg.cn/img_convert/6261c6e131b50419f5625475428742f5.png)

# VFIO-mdev

为了支持一个物理设备对应多个虚拟设备，但是该物理设备又不支持SRIOV的情况。

内核实现了一个虚拟设备（Mediated device）总线驱动模型，并在 VFIO 内核框架上进行了扩展，增加了对 mdev 这类虚拟设备的支持（mdev bus driver），可以从 mdev 设备驱动定义的虚拟设备接口来透传设备。这样，比如，当需要将一个 PCI 设备的 bar 空间作为资源切分的话，通过实现合适的 mdev 设备驱动，就可以将 bar 空间以 4KB（页面大小）为粒度，分别透传给不同虚机使用。


![96233416a17ad7590866112b9eb8778e.png](https://img-blog.csdnimg.cn/img_convert/96233416a17ad7590866112b9eb8778e.png)

# vDPA

vDPA 的全称是 Virtio Data Path Acceleration，它表示一类设备：这类设备的数据面处理是严格遵循 Virtio 协议规范的，即驱动和设备会按照第三节提到的 Virtio 通信流程来进行通信，但控制路径不一定会遵循 Virtio 协议

这个技术框架本质上和 VFIO-mdev 类似，也实现了一个虚拟设备（vDPA device）总线驱动模型，和 VFIO-mdev 不同的是，通过 vDPA 框架虚拟出来的设备，既可以给虚机使用，又可以直接从宿主机（比如：容器）进行访问

![eb2ca8541213b189640a249b8ed0060f.png](https://img-blog.csdnimg.cn/img_convert/eb2ca8541213b189640a249b8ed0060f.png)