# SimplygonAPI

## Processor Setting

> Simplygon 公开了几个设置，可以配置到可用的处理器上

### Mapping image settings

一些基本的设置，如生成的贴图的尺寸，是否生成新的纹理，以及如何产生。

- MaterialSettings

  - Texture Width And Height

    输出纹理的宽和高

### Bone reduction settings
Simplygon 能够通过将顶点重新链接到不同骨骼来减少动画中使用的骨骼数量。用户可以通过这个选项来选择每个顶点允许的最大骨骼数量，也可以限制场景中使用的骨骼总数。Simplygon可以自动检测哪些骨头最适合被移除，或者用户可以使用选择集手动选择保留或者移除哪些骨头

### Visibility settings
可以通过选定集或者选定子集来确定场景的可见性。比如用两个相机在不同的视角作为选择集，以这两个视角为可见性剔除不可见的部分。或者使用几何图形作为选择集，其顶点在内部转换为相机用来剔除。


### Repair settings

焊接一些有缝隙的三角形。


### Normal calcuation settings

在减面后，可以在新的几何上生成新的法线。该设置控制新法线的计算方式

### Geometry culling settings

控制裁剪平面的裁剪设置。

### Vertex weight settings

simplygon 允许用户通过在几何图形的顶点上设置权重来影响建面过程。这个设置对象配置什么时候应用顶点权重以及如何简单地解释顶点权重数据

### Reduction Settings
- Reduction Target Triangle Ratio
	减面的比例0-1。
	
- Reduction Target Triangle Ratio Enable
	这个控制上面那个，多此一举
	
- Reduction Target Triangle Count Enable
	控制下面那个是否开启
	
- Reduction Target Triangle Count
	控制生成的三角形的数量不能超过多少 1-nef
	
- Reduction Target Max Deviation
	减面后的几何和原始几何表面最大偏差，数值越高新建的模型的拓扑结构越不像原始物体
	
- Reduction Target On Screen Size
	根据物体在屏幕上占据的像素大小来计算减面
	
-  Reduction Target Stop Condition
	减面停止的标准，指以上几个选项哪个作为停止标准。Any选项是任意一个目标达成就停止，All是所有目标达到才停止。
	
-  Reduction Performance Mode
	性能选择，高性能下有一些特性不支持
	
-  Reduction Heuristics

  Fast模式下，三角数量和三角形占据屏幕像素这两个指标匹配的不是很精准。

- Reduction Importance

  保持拓扑结构的重要性，该选项会严重影响生成的新模型的拓扑结构和顶点数。不受减面比例的影响

- Material Improtance

  不同材质三角形边界在减面的时要考虑的比重。

- Texture Improtance

  UV在边界的重要性

- Shading Importance

  法线，锐利的边缘，以及法线插值在减面过程中重要性

- Group Importance

  不同sub-geometries之间

- 类似的还有颜色

- Edge set

  用户所选边的重要性，需要添加一个名为SgEdgeSelectionSet

- Skinning Importance

- Outward / Inward 

  允许几何向外或者向内扩展的程度

- Keep Symmetry

  是否保持对称，需要保证UV坐标也是对称的。

## Material casting

