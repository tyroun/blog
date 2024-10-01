{% raw %}

## 1 总体软件框架

![img](https://upload.wikimedia.org/wikipedia/commons/c/c2/Linux_Graphics_Stack_2013.svg)

![File:The Linux Graphics Stack and glamor.svg](../image/GPU笔记/800px-The_Linux_Graphics_Stack_and_glamor.svg.png)

### 2 GPU 驱动内容

#### 2.1 硬件架构

##### 整体架构

下图就是 Bifrost 架构，Shader Core 就相当于 NVIDIA 的 SM，与 NVIDIA 不同的是，Mali 的核心是可配置的，生产商可以根据需求自行设计自己的核数。同样的，各个 core 共享 L2
cache，通过一个类似总线的 GPU Fabric 相连。

![img](../image/GPU笔记/1620.png)

##### Shader Core 架构

对于每个 Shader Core 的架构如下。其中 Execution Engine（以下简写为 EE）就类似 NVIDIA 的 SP，但是不同的是，每个核中的 EE 数量很少。

![img](../image/GPU笔记/1620-16657287549765.png)

主要单元有：

1. Load/store unit 用于处理所有的内存的读写（除了纹理内存），包括 16KB L1 data cache.
2. Varying unit 这是一个专门为运算单元加速的单元。
3. Texture unit 这个单元是用来访问纹理内存的。
4. ZS & blend unit 适用于某些特定的 OpenGL ES 的操作。

##### Execution Engine

下图就是主要的架构，每个计算单元能够承载 4 个线程（在 G76 中可以承载 8 个线程）操作，也就是说对于 mali GPU 的 warp 大小是变化的，这 warp 对于内存还有什么调度都是相同的。

![img](../image/GPU笔记/1620-16657287549776.png)

{% endraw %}
