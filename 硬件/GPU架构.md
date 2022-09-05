# GPU架构
## 经典的架构
### NVidia Tesla架构
<image src=https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906000754905-1644664290.png >

Tesla微观架构总览图如上。下面将阐述它的特性和概念：
- 拥有7组TPC（Texture/Processor Cluster，纹理处理簇）
- 每个TPC有两组SM（Stream Multiprocessor，流多处理器）
- 每个SM包含：
    - 6个SP（Streaming Processor，流处理器）
    - 2个SFU（Special Function Unit，特殊函数单元,sin,cos）
    - L1缓存、MT Issue（多线程指令获取）、C-Cache（常量缓存）、共享内存
- 除了TPC核心单元，还有与显存、CPU、系统内存交互的各种部件。
### NVidia Fermi架构
<image src=https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906000823651-405733394.png >

- 拥有16个SM
- 每个SM
    - 2个Warp（线程束）
    - 两组共32个Core
    - 16组加载存储单元（LD/ST）
    - 4个特殊函数单元（SFU）
- 每个Warp
  - 16个Core
  - Warp编排器（Warp Scheduler）
  - 分发单元（Dispatch Unit）
- 每个Core
  -  1个FPU（浮点数单元）
  -  1个ALU（逻辑运算单元）
### NVidia Maxwell架构

<image src=https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906000959635-66123867.png >

采用了Maxwell的GM204，拥有4个GPC，每个GPC有4个SM，对比Tesla架构来说，在处理单元上有了很大的提升。

### NVidia Turing架构

<image src =https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001056361-789565826.png>

上图是采纳了Turing架构的TU102 GPU，它的特点如下：
- 6 GPC（图形处理簇）
- 36 TPC（纹理处理簇）
- 72 SM（流多处理器）
- 每个GPC有6个TPC，每个TPC有2个SM
- 4,608 CUDA核
- 72 RT核
- 576 Tensor核
- 288 纹理单元
- 12x32位 GDDR6内存控制器 (共384位)

单个SM的结构图如下：
<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001113377-1820574161.png>

每个SM包含:
- 64 CUDA核
- 8 Tensor核
- 256 KB寄存器文件

## GPU运行机制
下图是Fermi架构的运行机制总览图:
<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001159387-965011107.png>

从Fermi开始NVIDIA使用类似的原理架构，使用一个Giga Thread Engine来管理所有正在进行的工作，GPU被划分成多个GPCs(Graphics Processing Cluster)，每个GPC拥有多个SM（SMX、SMM）和一个光栅化引擎(Raster Engine)，它们其中有很多的连接，最显著的是Crossbar，它可以连接GPCs和其它功能性模块（例如ROP或其他子系统）。

程序员编写的shader是在SM上完成的。每个SM包含许多为线程执行数学运算的Core（核心）。例如，一个线程可以是顶点或像素着色器调用。这些Core和其它单元由Warp Scheduler驱动，Warp Scheduler管理一组32个线程作为Warp（线程束）并将要执行的指令移交给Dispatch Units。

GPU中实际有多少这些单元（每个GPC有多少个SM，多少个GPC ......）取决于芯片配置本身。例如，GM204有4个GPC，每个GPC有4个SM，但Tegra X1有1个GPC和2个SM，它们均采用Maxwell设计。SM设计本身（内核数量，指令单位，调度程序......）也随着时间的推移而发生变化，并帮助使芯片变得如此高效，可以从高端台式机扩展到笔记本电脑移动。

<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001212393-1848942244.png>

如上图，对于某些GPU的单个SM，包含:

- 32个运算核心 （Core，也叫流处理器Stream Processor）
- 16个LD/ST（load/store）模块来加载和存储数据
- 4个SFU（Special function units）执行特殊数学运算（sin、cos、log等）
- 128KB寄存器（Register File）
- 64KB L1缓存
- 全局内存缓存（Uniform Cache）
- 纹理读取单元
- 纹理缓存（Texture Cache）
- PolyMorph Engine：多边形引擎负责属性装配（attribute Setup）、顶点拉取(VertexFetch)、曲面细分、栅格化（这个模块可以理解专门处理顶点相关的东西）。
- 2个Warp Schedulers：这个模块负责warp调度，一个warp由32个线程组成，warp调度器的指令通过Dispatch Units送到Core执行。
- 指令缓存（Instruction Cache）
- 内部链接网络（Interconnect Network）

## GPU逻辑管线
下面将以Fermi家族的SM为例，进行逻辑管线的详细说明。

<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001228274-379363267.png>

1. 程序通过图形API(DX、GL、WEBGL)发出drawcall指令，指令会被推送到驱动程序，驱动会检查指令的合法性，然后会把指令放到GPU可以读取的Pushbuffer中。
2. 经过一段时间或者显式调用flush指令后，驱动程序把Pushbuffer的内容发送给GPU，GPU通过主机接口（Host Interface）接受这些命令，并通过前端（Front End）处理这些命令。
3. 在图元分配器(Primitive Distributor)中开始工作分配，处理indexbuffer中的顶点产生三角形分成批次(batches)，然后发送给多个PGCs。这一步的理解就是提交上来n个三角形，分配给这几个PGC同时处理。
> 以上可以说明，一次提交的DC，GPU会将模型分成多个三角形批次分给不同的PGCS并行处理。
<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906000842367-1857714844.png>

4. 在GPC中，每个SM中的Poly Morph Engine负责通过三角形索引(triangle indices)取出三角形的数据(vertex data)，即图中的Vertex Fetch模块。
5. 在获取数据之后，在SM中以32个线程为一组的线程束(Warp)来调度，来开始处理顶点数据。Warp是典型的单指令多线程（SIMT，SIMD单指令多数据的升级）的实现，也就是32个线程同时执行的指令是一模一样的，只是线程数据不一样，这样的好处就是一个warp只需要一个套逻辑对指令进行解码和执行就可以了，芯片可以做的更小更快，之所以可以这么做是由于GPU需要处理的任务是天然并行的。
> 在一个PGC内，以Warp为单位由多个线程共同处理这一组的顶点。
6. SM的warp调度器会按照顺序分发指令给整个warp，单个warp中的线程会锁步(lock-step)执行各自的指令，如果线程碰到不激活执行的情况也会被遮掩(be masked out)。被遮掩的原因有很多，例如当前的指令是if(true)的分支，但是当前线程的数据的条件是false，或者循环的次数不一样（比如for循环次数n不是常量，或被break提前终止了但是别的还在走），因此在shader中的分支会显著增加时间消耗，在一个warp中的分支除非32个线程都走到if或者else里面，否则相当于所有的分支都走了一遍，线程不能独立执行指令而是以warp为单位，而这些warp之间才是独立的。
7. warp中的指令可以被一次完成，也可能经过多次调度，例如通常SM中的LD/ST(加载存取)单元数量明显少于基础数学操作单元。
8. 由于某些指令比其他指令需要更长的时间才能完成，特别是内存加载，warp调度器可能会简单地切换到另一个没有内存等待的warp，这是GPU如何克服内存读取延迟的关键，只是简单地切换活动线程组。为了使这种切换非常快，调度器管理的所有warp在寄存器文件中都有自己的寄存器。这里就会有个矛盾产生，shader需要越多的寄存器，就会给warp留下越少的空间，就会产生越少的warp，这时候在碰到内存延迟的时候就会只是等待，而没有可以运行的warp可以切换。
<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001241355-608845528.png>

9.  一旦warp完成了vertex-shader的所有指令，运算结果会被Viewport Transform模块处理，三角形会被裁剪然后准备栅格化，GPU会使用L1和L2缓存来进行vertex-shader和pixel-shader的数据通信。
10. 接下来这些三角形将被分割，再分配给多个GPC，三角形的范围决定着它将被分配到哪个光栅引擎(raster engines)，每个raster engines覆盖了多个屏幕上的tile，这等于把三角形的渲染分配到多个tile上面。也就是像素阶段就把按三角形划分变成了按显示的像素划分了。
<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001300961-1313843419.png>

11. SM上的Attribute Setup保证了从vertex-shader来的数据经过插值后是pixel-shade是可读的。
12. GPC上的光栅引擎(raster engines)在它接收到的三角形上工作，来负责这些这些三角形的像素信息的生成（同时会处理裁剪Clipping、背面剔除和Early-Z剔除）。
13. 32个像素线程将被分成一组，或者说8个2x2的像素块，这是在像素着色器上面的最小工作单元，在这个像素线程内，如果没有被三角形覆盖就会被遮掩，SM中的warp调度器会管理像素着色器的任务。
14. 接下来的阶段就和vertex-shader中的逻辑步骤完全一样，但是变成了在像素着色器线程中执行。 由于不耗费任何性能可以获取一个像素内的值，导致锁步执行非常便利，所有的线程可以保证所有的指令可以在同一点。
15. 最后一步，现在像素着色器已经完成了颜色的计算还有深度值的计算，在这个点上，我们必须考虑三角形的原始api顺序，然后才将数据移交给ROP(render output unit，渲染输入单元)，一个ROP内部有很多ROP单元，在ROP单元中处理深度测试，和framebuffer的混合，深度和颜色的设置必须是原子操作，否则两个不同的三角形在同一个像素点就会有冲突和错误。
> 以上说明不一定是一个物体执行一次运算。

### GPU技术要点
1. SIMD 单指令多数据，一个指令可以处理多维向量（一般是4D的数据）。比如有一下的shader指令:
  ``` C#
  float4 c = a + b; // a, b都是float4类型
  ```
对于没有SIMD的处理单元，需要四条指令将四个Float值相加，汇编伪代码如下:
``` C# 
ADD c.x, a.x, b.x
ADD c.y, a.y, b.y
ADD c.z, a.z, b.z
ADD c.w, a.w, b.w
```
但有了SIMD技术，只需一条指令即可处理完：
``` C#
SIMD_ADD c, a, b
```

2. SIMT 单指令多线程
   可对GPU单个SM中的多个Core同时处理同一指令，并且每个Core存取的数据可以是不同的。
3. co-issue
   co-issue是为了解决SIMD运算单元无法充分利用的问题。例如下图，由于float数量的不同，ALU利用率从100%依次下降为75%、50%、25%。

   <image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001418746-831705203.png>

   为了解决着色器在低维向量的利用率低的问题，可以通过合并1D与3D或2D与2D的指令。例如下图，DP3指令用了3D数据，ADD指令只有1D数据，co-issue会自动将它们合并，在同一个ALU只需一个指令周期即可执行完。

   但是，对于向量运算单元（Vector ALU），如果其中一个变量既是操作数又是存储数的情况，无法启用co-issue技术：

   于是标量指令着色器（Scalar Instruction Shader）应运而生，它可以有效地组合任何向量，开启co-issue技术，充分发挥SIMD的优势。
4.  Early-Z(有一个专题)

早期的GPU的渲染管线的深度测试是在像素着色器之后才执行（下图），这样会造成很多本不可见的像素执行了耗性能的像素着色器计算。
<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001513349-915323989.png>

将深度测试提前可以减少不可见片元的开销。

<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001521348-550843794.png>

Early-Z技术可以将很多无效的像素提前剔除，避免它们进入耗时严重的像素着色器。Early-Z剔除的最小单位不是1像素，而是像素块（pixel quad，2x2个像素)

以下情况会导致Early-Z失效:
- **开启Alpha Test** 
- **开启Tex Kill**
- **开启MSAA**

5. 统一着色器架构
在早期的GPU，顶点着色器和像素着色器的硬件结构是独立的，它们各有各的寄存器、运算单元等部件。这样很多时候，会造成顶点着色器与像素着色器之间任务的不平衡。对于顶点数量多的任务，像素着色器空闲状态多；对于像素多的任务，顶点着色器的空闲状态多（下图）。

<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001553857-826068031.png>

于是，为了解决VS和PS之间的不平衡，引入了统一着色器架构（Unified shader Architecture）。用了此架构的GPU，VS和PS用的都是相同的Core。也就是，同一个Core既可以是VS又可以是PS。

6. 像素块

上一节步骤13提到：
> 32个像素线程将被分成一组，或者说8个2x2的像素块，这是在像素着色器上面的最小工作单元，在这个像素线程内，如果没有被三角形覆盖就会被遮掩，SM中的warp调度器会管理像素着色器的任务。

也就是说在像素着色器中，会将相邻的四个像素作为不可分隔的一组，送入同一个SM内4个不同的Core。

这种设计虽然有其优势，但同时，也会激化过绘制（Over Draw）的情况，损耗额外的性能。比如下图中，白色的三角形只占用了3个像素（绿色），按我们普通的思维，只需要3个Core绘制3次就可以了。

<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001651573-804337756.png>

但是，由于上面的3个像素分别占据了不同的像素块（橙色分隔），实际上需要占用12个Core绘制12次（下图）。

<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001659282-1154417522.png>

## GPU资源机制
### 内存架构

部分架构的GPU与CPU类似，也有多级缓存结构:寄存器，L1,L2缓存，GPU显存，系统显存。

<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001709237-945454718.png>

### GPU Context和延迟

由于SIMT技术的引入，导致很多同一个SM内的很多Core并不是独立的，当它们当中有部分Core需要访问到纹理、常量缓存和全局内存时，就会导致非常大的卡顿（Stall）。

例如下图中，有4组上下文（Context），它们共用同一组运算单元ALU。

<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001735780-1606282901.png>

假设第一组Context需要访问缓存或内存，会导致2~3个周期的延迟，此时调度器会激活第二组Context以利用ALU：

<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001744570-1497753939.png>

当第二组Context访问缓存或内存又卡住，会依次激活第三、第四组Context，直到第一组Context恢复运行或所有都被激活：

这部分和CPU的并行很类似。

### CPU-GPU异构系统

根据CPU和GPU是否共享内存，可分为两种类型的CPU-GPU架构：

<image src= https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001850363-356121869.png>

上图左是分离式架构，CPU和GPU各自有独立的缓存和内存，它们通过PCI-e等总线通讯。这种结构的缺点在于 PCI-e 相对于两者具有低带宽和高延迟，数据的传输成了其中的性能瓶颈。目前使用非常广泛，如PC、智能手机等。

上图右是耦合式架构，CPU 和 GPU 共享内存和缓存。AMD 的 APU 采用的就是这种结构，目前主要使用在游戏主机中，如 PS4。

在存储管理方面，分离式结构中 CPU 和 GPU 各自拥有独立的内存，两者共享一套虚拟地址空间，必要时会进行内存拷贝。对于耦合式结构，GPU 没有独立的内存，与 GPU 共享系统内存，由 MMU 进行存储管理。

### GPU资源管理模型

下图是分离式架构的资源管理模型：
<image src = https://img2018.cnblogs.com/blog/1617944/201909/1617944-20190906001903861-1080252910.png>

- MMIO（Memory Mapped IO）
  - CPU与GPU的交流就是通过MMIO进行的。CPU 通过 MMIO 访问 GPU 的寄存器状态。
  - DMA传输大量的数据就是通过MMIO进行命令控制的。
  - I/O端口可用于间接访问MMIO区域，像Nouveau等开源软件从来不访问它。
- GPU Context
  - GPU Context代表了GPU计算的状态。
  - 在GPU中拥有自己的虚拟地址。
  - GPU 中可以并存多个活跃态下的Context。
- GPU Channel
  - 任何命令都是由CPU发出。
  - 命令流（command stream）被提交到硬件单元，也就是GPU Channel。
  - 每个GPU Channel关联一个context，而一个GPU Context可以有多个GPU channel。
  - 每个GPU Context 包含相关channel的 GPU Channel Descriptors ， 每个 Descriptor 都是 GPU 内存中的一个对象。
  - 每个 GPU Channel Descriptor 存储了 Channel 的设置，其中就包括 Page Table 。
  - 每个 GPU Channel 在GPU内存中分配了唯一的命令缓存，这通过MMIO对CPU可见。
  - GPU Context Switching 和命令执行都在GPU硬件内部调度。
- BO
  - Buffer Object (BO)，内存的一块(Block)，能够用于存储纹理（Texture）、渲染目标（Render Target）、着色代码（shader code）等等。


### 显像机制

垂直同步：
GPU等一个FB渲染，提交指令才会交换缓冲.

## 最后一些作者的验证

1. 处理2x2的像素块时，那些未被图元覆盖的像素着色器线程将被标记为gl_HelperThreadNV = true，它们的结果将被忽略，也不会被存储，但可辅助一些计算，如导数dFdx和dFdy