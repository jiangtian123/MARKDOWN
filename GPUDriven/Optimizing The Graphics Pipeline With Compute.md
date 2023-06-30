# 优化Pipeline计算

如何通过减少渲染那么多的三角形提高三角形的渲染速度

## 一些缩写所代表的意思，后文中会经常用到

![](D:\Users\Administrator\Desktop\内存测试\名词缩写.jpg)

## 分析目标平台的三角形发出速率

每个shader engines 在一个时钟周期内发出一个三角形，PC端较新的AMD显卡有四个shader engines，可以发出四个三角形每个时钟周期。

将计算单元(compute units)的数量 * 64 * ALU数量 * 2 即可得到一个时钟周期内ALU操作执行的数量。

使用ALU操作执行的数量 / engines的数量，可以得到一个时钟周期内，每个三角形中可以执行的ALU操作的数量

将每个三角形的ALU操作数 / 每个时钟周期ALU 的操作数，可以得到一个最终的指令上限。

乐观点假设**Rasteriaer**每个时钟周期能处理两个三角形，实际上xb1是0.9个。因为从VGT->PA使用FIFO队列，所以在4096个时钟周期内填满FIFO队列，才不会使Rasterizer等待。

作者在得到每个时钟周期内接近 两个Prims的效果时有以下特征：

- VS 没有读任何东西
- VS仅输出SV_Position
- 每个VS仅输出0.0，都被剔除了

大部分的引擎都会在CPU端做剔除，再在GPU上refine。但是因为PCIE的传输速率低，所以我们会做一些GPU上的剔除，本篇主要讲cluster/triangleculling。

CS处理mesh带来的好处。最主要的思想就是把drawcall看作data，这些GPU生成的数据可以pre-built，cached，reused。

### Culling Overview

- Scenen包括
  - 网格集合
  - 特定的视角
  - camera light 等

将场景中的网格合并成一个Batch，即网格拥有相同的vertex index 步长，共享shader。

当然，batch划分为mesh section，即一次绘制需要的内容：

- vertex buffer
- index buffer
- primitive count
- 等

在swap中处理最佳的量：

- AMD中一个**wavefront(warp)**有64个线程
- 一个culling Thread处理一个三角形
- 一个work item处理256个三角形

![](D:\Users\Administrator\Desktop\内存测试\culling Overview.jpg)

### Maping Mesh ID to MultiDraw ID

