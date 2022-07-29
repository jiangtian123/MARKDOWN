# UniversalRenderPipeline
> 该类为真正去驱动各个Pass做渲染的类，继承**RenderPipeline** ，每帧都会调用重写的Render方法。
## 初始化及构造函数
- 初始化的时候会定义一些全局控制参数。如最大阴影Bias，RenderScal，AdditionalLight。

- 构造函数
   1. 会生成一个**UniversalRenderPipelineGlobalSettings** 。
   2.  SetSupportedRenderingFeatures()生成一个SupportedRenderingFeatures()对象。
   3.  初始化屏幕大小。
   4.  获取是否支持SRPBatcher
   5.  获取MSAA的采样数和设置MSAA
   6.  设置shader的全局名称“UniversalPipeline”，在URP管线下，tag不为空，且不是该值的，则赋予错误材质。
   7.  设置GI
## Render（）
