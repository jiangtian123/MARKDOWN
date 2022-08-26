# 12.1URP与10.7
## Renderer
1. DepthPrimingMode
   12.1多了一个枚举类型，当需要深度图的时候会用到这个类型。
2. k_DepthStencilBufferBits
   这个变量多了一个24位的选项
3. 没有colorAttachment了
   取代的是m_ColorFrontBuffer
4. m_PrimedDepthCopyPass
   这个pass是个CopyDepthPass，执行Event为:AfterRenderingPrePasses
5. 贴图
   - _CameraColorTexture没有了，换成了_CameraColorAttachment，但是类型为RenderTargetBufferSystem
   - _AfterPostProcessTexture没有了
   - _InternalGradingLut无了
6. RenderTargetBufferSystem多了一个这个类
## 添加Pass的变化
### requiresDepthPrepass
```C# 
    requiresDepthPrepass =((cameraData.requiresDepthTexture || renderPassInputs.requiresDepthTexture || m_DepthPrimingMode == DepthPrimingMode.Forced) ||(applyPostProcessing && cameraData.postProcessingRequiresDepthTexture))&&!CanCopyDepth(ref renderingData.cameraData);

    bool CanCopyDepth(ref CameraData cameraData)
        {
            bool msaaEnabledForCamera = cameraData.cameraTargetDescriptor.msaaSamples > 1;
            bool supportsTextureCopy = SystemInfo.copyTextureSupport != CopyTextureSupport.None;
            bool supportsDepthTarget = RenderingUtils.SupportsRenderTextureFormat(RenderTextureFormat.Depth);
            bool supportsDepthCopy = !msaaEnabledForCamera && (supportsDepthTarget || supportsTextureCopy);

            bool msaaDepthResolve = msaaEnabledForCamera && SystemInfo.supportsMultisampledTextures != 0;

            // copying depth on GLES3 is giving invalid results. Needs investigation (Fogbugz issue 1339401)
            if (IsGLESDevice())
                return false;

            return supportsDepthCopy || msaaDepthResolve;
        }
    该方法，只要开启了MSAA且采样数>1，或者没有开Msaa但是显卡支持深度图格式或者支持深度图拷贝，则为true。或者只要检测到是es2或者es3就会返回false
    requiresDepthPrepass |= isSceneViewCamera;
    requiresDepthPrepass |= isGizmosEnabled;
    requiresDepthPrepass |= isPreviewCamera;
    requiresDepthPrepass |= renderPassInputs.requiresDepthPrepass;
    requiresDepthPrepass |= renderPassInputs.requiresNormalsTexture;
```
总结一下:
1. 被动开启
   相机的DeptpTexture为On，RenderFeature中有用到深度图，Renderer设置中的Depth Priming Mode为Forced，开启了后处理，且后处理栈中有用到运动模糊和深度的。以上只要满足一个，且显卡不支持深度图拷贝，没有开启MSAA和API为OpenglE3的就会开启PreDepthPass。

2. 主动开启
   - m_DepthPrimingMode == DepthPrimingMode.Forced。
   - requiresDepthPrepass |= renderPassInputs.requiresDepthPrepass;
   - requiresDepthPrepass |= renderPassInputs.requiresNormalsTexture;

