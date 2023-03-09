# Mesh Shader
- 提出的原因：
	1. Gpu在进行VS之前，Primitive Distributor 会每帧去显存里拿取顶点数据然后重新组织（按照Batch重新生成顶点索引的索引），分配warp去处理。即使模型的拓扑结构没有改变过。
	
	2. 模型以三角面级别做剔除
	
	3. 顶点复用率超过3，顶点就没有办法装满整个Warp。  
	   <img src="https://pic3.zhimg.com/v2-9b947592db07bdf5781c70326455b212_r.jpg" style="zoom:50%;" />
	
## 新的解决手段
以往的时候，我们使用CS来简介绘制，但是还会走一遍PD，如果能直接把CS的输出结果送给光栅化器就好了。于是，便有了MeshShader。

也就是说，现代GPU硬件提供了一种可以让我们自定义几何阶段的操作，完全替代了以前的顶点获取分配，VS，曲面细分，Gs阶段。
## Mesh Shading Popeline
分两部分
1. Task Shader
	一个可编程单元，在workgroups工作组中操作，生成每个需要或者不需要的mesh shader工作组。
2. Mesh Shader
	在workgroups工作组中操作，可以生成新的图元。
	

与传统管线的区别:

![](https://developer.nvidia.com/blog/wp-content/uploads/2018/09/meshlets_pipeline.png)