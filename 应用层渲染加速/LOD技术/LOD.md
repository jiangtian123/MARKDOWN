# Unity-AutoLOD

Unity官方介绍说，使用这个可以牺牲一点内存来换取十倍的渲染性能。

## 动机

为了提高渲染画面的表现力，同时又能让性能撑得住。我们需要使用一些手段来降低CPU和GPU的工作量。例如：遮挡剔除，纹理图集，动静态合批，GPU实例化。使用不同层次细节的**LOD**是一个不错的技术。

如果普通的LOD那么可靠，我们也没必要写这篇文章了。

普通的LOD有以下问题：
- LOD的制作需要美术手动修改
- 需要对管线进行修改

## 生成过程

### AUTOLod.生成LOD

对原始模型生成LOD，可以自定义LOD级别。

###  ScenenLOD.管理带有LOD的物体
 SceneLOD有几个集合分别介绍一下都是什么

- **Dictionary<GameObject, Pose> m_SelectedObjectLastPose** 当前场景中已经处理过的每个物体在空间中的位置，朝向
- **HashSet< Renderer> m_ExcludedRenderers** 存当前帧所有renderer中需要被排除的物体
- **List< Renderer> m_FoundRenderers**一次性把场景中的Renderer Load到这个集合中
- **HashSet< Renderer> m_ExistingRenderers**  当前场景Volume中已经有的renderer
- **HashSet< Renderer> m_AddedRenderers** 当前场景中新添加的物体
- **HashSet< Renderer> m_RemovedRenderers ** 当前场景中在这一帧被删除的物体的物体
#### UpdateOctree

从场景中Load第一个带有LodVolume的物体，随后遍历场景中所有带Renderer的物体，然后按照物体的种类进行剔除。

- 如果物体的Layer是HLOD的，剔除。因为带有Hlod layer的物体处理过的。

- 物体Mesh不是三角形的剔除

- 父物体没有LodGroup组件的不剔除

  原因是如果物体的父物体没有lodGroup，说明它是独立的渲染物体，本身不存在Lod。而HLOD是在已经生成LOD的基础上运行的

这个方法是用来管理场景的Octree的，对于删除的物体从树中剔除，新加的符合条件的添加到树中。


#### OnSelectionChanged()

当选择的状态发生变换时，如从未选择任意一个物体到选中一个或多个物体时调用该方法。用来更新选择物体的位置（如果有变化的话）并且更新它在树中的位置。

总的来说，ScenenLOD只负责检测场景中的数据发生改变时回调相应的方法来应对，真正执行场景 管理的地方时LODVolumes。

### LODVolume

当场景没有LodVolume的时候创建一个LodVolume作为根节点。并且添加一个LodVolume组件。所有处理过的Renderer都在这个组件里管理

核心方法介绍

####  AddRenderer

向Volume中添加新的带有LODGroup的Rendeer

首先是设置包围盒，如果是第一个物体，则整个Volume的包围盒用的是第一个的，随后再传进来再扩展包围盒。

```C++
bool first;
if(first)
{
    //取当前物体包围盒XYZ轴最长的那个作为新构建包围盒的长宽高，是一个立方体。
}
//判断是否需要扩展包围盒
//如果新添加的物体的包围盒长度为0或者中心就在Volume的包围盒中
bool WithinBounds(renderer, bounds)
{
    return Mathf.Approximately(bounds.size.magnitude, 0f) || bounds.Contains(r.bounds.center);
}
if(WithinBounds(renderer, bounds))
{
    //如果Volume还没有这个Rnederer，则添加进去
    //同时，如果Volume还没有分割（切块）,且保存的Renderers大于32个了，则分割一下。
    //同时，如果Volume还没有分割（切块），且保存的Renderers小于32个，将dirty置为True。
	//如果已经分割过，则判断该Renderer在哪个块
}
else
{
    //如果该物体不在这个包围盒的中心，且长度大于一，同时这个Volume没有分割。
    //将Volume扩大到两个包围盒刚好容下的位置。
}
```
#### UpdateHLODs()

对一个Volume及其子物体调用GenerateHLOD（）；

#### GenerateHLOD



#### Split()

调用时机: 当一个Volume的renderers多于32且还没有分割的时候，或者主动调用的时候。

一个大的划分为八个小的。
