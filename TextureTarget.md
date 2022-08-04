# 相机贴图的替换历史（包括后处理）
## renderer.setup阶段
### 如果在编辑器中给相机一个贴图，同时这个贴图的格式为Depth
则会直接配置相机的渲染纹理。包括color和depth为：  
**BuiltinRenderTextureType.CameraTarget**
然后将所有rendererFeature的pass加入队列中，同时将Opaque，SkyBox，Transparent一次加入pass队列。
>这个队列**m_ActiveRenderPassQueue** 之后还会按照event进行排序和分块。

添加完后，就会退出Renderer.setup这个方法了
### 如果相机没有配置targetTexture或者配置的targetTexture不是Depth
1. ColorTexture
    ```C#
    var createColorTexture = (rendererFeatures.Count != 0 && !isRunningHololens) && !isPreviewCamera;
    ```
    如果createColorTexture为ture，则配置ColorTexture为ForwardRenderer准备好的**m_CameraColorAttachment**，该贴图名为“_CameraColorTexture” 
2. OtherTexture
    包括CopyDepthTexture，NormalsTexture，CopyColorTexture

    URP准备了一个数据结构来判断是否需要以上贴图，这个数据结构还有一个isNeedPrePass
    ```C#
    private struct RenderPassInputSummary
        {
            internal bool requiresDepthTexture;
            internal bool requiresDepthPrepass;
            internal bool requiresNormalsTexture;
            internal bool requiresColorTexture;
        }
     private RenderPassInputSummary GetRenderPassInputs(ref RenderingData renderingData)
        {
            RenderPassInputSummary inputSummary = new RenderPassInputSummary();
            for (int i = 0; i < activeRenderPassQueue.Count; ++i)
            {
                ScriptableRenderPass pass = activeRenderPassQueue[i];
                bool needsDepth = (pass.input & ScriptableRenderPassInput.Depth) != ScriptableRenderPassInput.None;
                bool needsNormals = (pass.input & ScriptableRenderPassInput.Normal) != ScriptableRenderPassInput.None;
                bool needsColor = (pass.input & ScriptableRenderPassInput.Color) != ScriptableRenderPassInput.None;
                bool eventBeforeOpaque = pass.renderPassEvent <= RenderPassEvent.BeforeRenderingOpaques;

                inputSummary.requiresDepthTexture |= needsDepth;
                inputSummary.requiresDepthPrepass |= needsNormals || needsDepth && eventBeforeOpaque;
                inputSummary.requiresNormalsTexture |= needsNormals;
                inputSummary.requiresColorTexture |= needsColor;
            }

            return inputSummary;
        }
    ```
    以上是判断是否需要额外的pass的方法。也就是说，激活的pass中，只要有一个需要depth，color，normal，就会使**RenderPassInputSummary** 对应的变量变为ture。但是DepthperPass要求会严格一些。

    通过**CreateCameraRenderTarget()** 方法来配置相机的渲染贴图，最后通过两个参数来控制：


    - m_ActiveCameraColorAttachment
    - m_ActiveCameraDepthAttachment

    这个两个参数又通过两个Bool类型变量来控制值：

    - createColorTexture
    - createDepthTexture
    
    上诉两个Bool类型的变量只要有一个为True且相机为BaseCamera，则调用**CreateCameraRenderTarget()** 方法。

    