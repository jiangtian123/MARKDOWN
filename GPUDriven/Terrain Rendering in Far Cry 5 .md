# Terrain Rendering in Far Cry 5

- 全屏地形的渲染控制在 2ms内
- 地形与草，石头，植被等元素融合的更加逼真且融合起来简单

## Heightfield Rendering（高度场渲染）

一种地形渲染方案。

一张纹理的四个分量(x,y,z,w)

- xz代表在xz轴上的坐标
- y代表高度
- w可以是材质id

渲染一个面片，在VertexShader中，用xz取采样地形图，改变面片的y轴，并在fs中应用材质渲染地形。

在FarCry中

每**Tiles**是64m*64m的面积，FarGry称每个Tile为**Sector**。

地形是10km*10km的，一共160X160个Sector。

像素与像素之间的间隔的精度是**0.5m**。

<img src="D:\MyMakrDown\GPUDriven\Photo\2023817四-下午-063635.jpg" style="zoom:80%;" />

为了不以最高分辨率渲染每个Title(因为远处的地形不需要高精度),FarCry使用了一颗四叉树来管理地形的显示。

在根节点，10km x 10KM的地形为分割为了2km x 2km的块，并且常驻。感觉类似于多个地形分了多个四叉树

再将2km变长的方块划分为层级四叉树。

根据LOD级别和距离玩家的距离来流式加载**Tiles**。

最大分配约500常驻Tiles大小的内存。

每个四叉树节点保存的数据：

- 高度图
- 世界坐标空间下的normal map
- Albedo map

加载一个quad tree node的时候加载这些贴图。

贴图的格式如下：

- Heightmap (R16_UNORM,129 X 129)
- normal map (BC3,132 X 132)
- Baked albedo map (BC1,132 X 132，使用一位的alpha来表示地形上是否有洞)

## Terrain Rendering

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-上午-102223.jpg" style="zoom:80%;" />

地形渲染的步骤

### Stream Quad Tree

建立四叉树。以玩家位置为中心，选择LOD级别进行加载。

一个理想的加载状态如下图所示：

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-上午-103112.jpg" style="zoom:80%;" />

但是我们不能保证节点在任何时候都会被加载进来，这是一个理想的状态。实际的情况可能如下图：

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-上午-103321.jpg" style="zoom:80%;" />

### Traverse Quad Tree

从根节点开始单步，选择要渲染的节点。FarCry大概有6级Lod。

什么时候选择子节点

- 一个节点的子节点都被Loaded。
- 当前模块想要更高的LOD级别。

选中节点的级别后，再用相机进行剔除。

得到最终渲染的节点如下：

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-上午-104044.jpg" style="zoom:80%;" />

### Batch into Shading Groups

远处的节点和近处的节点可能使用不同的Shader渲染，最后一步我们将使用同一种Shader的节点进行合批，提交渲染。

最初上述工作的大部分内容都在CPU进行

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-上午-104430.jpg" style="zoom:80%;" />

现在随着GPU处理能力的提高，我们可以将大部分工作转移到GPU处理，只保留最开始的四叉树的加载。

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-上午-104534.jpg" style="zoom:80%;" />

## GPU Terrain

将大量工作交给GPU做的动力是什么呢：

- 数据只被GPU处理
- 减少CPU消耗
- 降低GPU消耗
  - 可以实现顶点级别的剔除工作
- 共用性
  - 使 Treeain数据可以共享于其他GPU systems
    - Trees，Rocks，Grass，other

### GPU Data Structures

- Terrain Quad  Tree
	- 持续存在 - 当loading/unloading 节点时更新
- Terrain Node List
  - 四叉树遍历生成的列表
  - 每帧更新
- Terrain Lod Map
  - 世界上每个Sector使用的几何LOD
  - 从Terrain Node List中每帧更新一次
- Visible Render Patch List
  - 经过剔除后的render node list
  - 每次渲染前更新

### GPU Data Structures存储方式

#### Terrain Quad Tree

一张带有mip-map的图，这张图大小为160 x 160 像素(因为游戏大小为160 x 160 个sectors)，有六级mip-map。每个像素代表一个节点。

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-上午-112331.jpg" style="zoom:80%;" />

那这张纹理存储了什么呢？

每个像素为16bit，存储的是指向另一个GPU 数据结构的索引。

指向的为一个简单的节点描述缓冲区，是一个array。

该缓冲区包含了最小/最大值的地形高度，以及节点用到的贴图在纹理图集中的位置。

Texture 是 R16_UINT。

node data 是 2uints。

有两个特殊数据

- 0xffff 意味着四叉树没有加载任何东西
- 0xfffe 意味着四叉树没有东西。

#### Terrain Node List

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-014307.jpg" style="zoom:80%;" />

保存了在做进一步可见性测试之前可能看见的Node数据。

他是通过遍历上一个四叉树数据结构来每帧更新的。

构建 **Terrain Node List**：

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-014943.jpg" style="zoom:80%;" />

使用CS根据LOD级别循环填充它，每个线程处理一个节点。

先使用四叉树所有最低级别LOD的节点填充一个临时缓冲 Temproary Buffer A。

接着遍历 Buffer A，如果能够更进一步，将它子节点加入到Buffer B。

如果不可以在分，则将该节点放入最终的 rendering Buffer。

遍历完 Buffer A后，还剩下两个Buffer：

- Rendering Buffer
- Buffer B

接下来遍历 Buffer B

同样，Buffer B 中可以再划分的，这时放入Buffer A，不可以再划分的 进入 Rendering Buffer。

这样循环遍历，我们就得到了一个 Terrain Node List，存储了不同LOD级别的可被渲染Node。

#### Terrain LOD Map

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-030842.jpg" style="zoom:80%;" />

每一帧，都会填充这样一个8 bit 的纹理，给每一个Sector标记上LOD级别。

需要这样的一个数据结构的唯一作用就是检测，不同LOD级别之间的Patch 怎么避免出现接缝。

只在相邻的Patch具有较低LOD的时候修改Patch顶点，通过填补0，可以忽略掉不需要的。

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-031511.jpg" style="zoom:80%;" />

利用上一步的List，根据节点的覆盖范围，再Map上标记Sector的LOD级别。

CS上有一些线程方面需要注意。

#### Visible Render Patch Lost

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-031816.jpg" style="zoom:80%;" />

**Patch**是最小的细分单位，直接伴随着一个Indirect Args 提交DC。

每个**Patch**是16 * 16 大小的网格。

通过创建的**Terrain Node List**来构建 patch list，并剔除不可见的patch。

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-032237.jpg" style="zoom:80%;" />

一个Node 可以划分为8*8个**Patch**。

针对每个Patch做以下处理：

- Frustum Cull
- Occlusion buffer cull
- Back Face Cull
- Calculate and pack LOD transitions into the patch data

## GPU Culling

就是GPU PIPE Line那一套，在其他地方写过了。

## LOD Mesh Stitching

<img src="D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-033640.jpg" style="zoom:80%;" />

大世界地形解封问题。

![](D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-033849.jpg)

对于相邻的两块Patch，但是LOD级别不一样，会导致边缘出现开裂的现象。

通过读取在Culling阶段的数据，我们可以知道一个Patch合适连接到一个LOD级别较低的Patch上。

在vertex shader 中，我们将较高LOD级别的Patch 的顶点焊接起来如下图：

![](D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-034240.jpg)

![](D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-034258.jpg)

最后变成下图这样：

![](D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-034329.jpg)

有时LOD级别的下降很快，我们可以一次焊接多个顶点：

![](D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-034426.jpg)

对于带洞的地形，使用 1 bit的 Alpha通道存储在 albedo map中。在vertex shader 中，如果该点是洞，则丢弃(通过输出 nan值)

由于删除一个顶点，会导致它链接的三角形都会被剔除，所以洞的大小要是地形分辨率的一半。

![](D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-034844.jpg)

![](D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-034853.jpg)

## Terrain Shading

shading 的方法来自于 ： 2017 GDC talk on Ghost Recon Wildlands。

每个节点有带有自己需要贴图的索引，除了之前提到的贴图，还有几种：

![](D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-042920.jpg)

- Heightmap (R16_UNORM, 129x129) 
- World space normal map (we can assume positive z and also pack smoothness and  specular occlusion) (BC3, 132x132)
-  Albedo map (BC1, 132x132)
-  Color modulation map (BC1, 132x132) 
- Splat map (R8_UNORM, 65x65)
- Patch cone texture (BC3, 8x8)

讲一下Terrain Splat Map

![](D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-043847.jpg)

对于远处的shading 可以使用直接且廉价的手段，使用 albedo/normal。

对于近处的shading，我们需要使用 **splat map**。

大世界如果只使用一种材质，将很单调。所以，splat map 保存了材质信息。

地形前景渲染如何做：

![](D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-044417.jpg)

当我们想渲染一个地形像素时：

- 通过tile 获取这个像素的local position
- 使用local position 去采样 splat map 获取材质ID
- 通过材质ID获取材质描述信息（包括要采样的贴图信息）
-  通过对VU进行操作，丰富地形

记得做混合

![](D:\MyMakrDown\GPUDriven\Photo\2023818五-下午-044819.jpg)

## Virtual Texture

这一块在 Far Cry4 有介绍。



## Cliff Shading

