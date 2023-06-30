# 基于Unity3D的大地形绘制学习

地形渲染的难点集中在：

- 提交绘制

  当地形很大且复杂（高山低谷）时，常需要用数个Pass绘制同一片地形，如一个pass绘制地面，一个山一个河流。这就会导致Drawcall爆炸，造成CPU消耗难以控制。

- LOD控制

  大地形的视距往往很夸张，而远处的物体很模糊，如果采用高质量的顶点来绘制，就会造成顶点shader的消耗巨大，所以需要有LOD技术来控制显示的精度。

- 流式加载资源

  资源要秉承着，看不见的就从内存中卸载出去，看得见时再加载进来的原则。

按照**MaxWellGeng**大佬的文章来记录学习过程，分以下部分

1. 开放大地形的绘制方法
2. 显存资源的加载与卸载
3. 资源的序列化与反序列化
4. 场景自动分块工具
5. 开放世界自动生成工具

## 从GPUDrivenPipeline开始（siggraph2015）理论

解放CPU，由GPU承担更多的提交和剔除工作。

### Motivation（动机）

- 巨量的建筑导致CPU端GC的开销剧增
- 无缝的建筑结构
- 大量的人物

### Mesh Cluster Renderer

- 将模型重新拓扑，64个顶点为一个Cluster
- VS从buffer中获取顶点
- 在GPU中根据Cluster进行剔除到一个list，然后渲染
- 任意数量的Mesh一个Drawcall
- Gpu根据cluster bounds进行剔除
- 更快的vertex获取
- 按照Cluster的深度排序
- 使用triangle strips带来的问题（triangle strips 是不存储重复顶点的三角形）
  - 将模型拆分成Cluster时，每个Instance要退化三角形进行补齐，每个Mesh的最后Cluster也需要退化三角形，增加了内存消耗
  - 不确定的 Cluster 顺序
- 使用MultiDrawIndexedInstancedIndirect
  - 动态管理IndexBuffer
  - 不需要退化三角形
  - 也可以进行Cluster排序

### Pipeline

![](D:\Users\Administrator\Desktop\内存测试\GpuDrivenPpeline.jpg)

- CPU端
  - 粗略的视锥体剔除，如四叉树剔除
  - 构建合批数据更新Instance数据
  - 合批DrawCall
- GPU端
  - 根据Instance的包围盒进行视锥体和遮挡剔除
  - 将Instance拆分成Chunk（什么是Chunk啊？）
  - Cluster剔除（视锥，遮挡，三角形，背面）
  - Index buffer的压缩，即将空的Instance剔除，Buffer里只留可见的
  - 调用ExecuteIndirect没什么好说的

### 解决一些问题

1. 什么是Instance？
2. 什么是Cluster？
3. 什么是Trunk?