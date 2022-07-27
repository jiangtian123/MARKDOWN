# LearningURP
## 管线驱动
### 前言
> 管线驱动阶段主要做了两件事，一是设置各种全局数据，二是驱动各个Pass去做真正的渲染
### URP入口——UniversalRenderPipelineAsset
该类继承**RenderPipelineAsset**，负责获取管线对象，驱动真正的管线去做渲染。设置一些全局变量。同时作为参数给**UniversalRenderPipeline**。该类还做了一些对BuildIn管线的支持，忽略不看。
通过重写**CreatePipeline()** 方法创建一个管线对象



### UniversalRenderPipeline类
> 该类继承至**RenderPipeline**，重写的**Render()** 方法为整个渲染的入口函数。
#### UniversalRenderPipeline() 这个构造函数配置一些全局参数:  
1. 设置是否支持<font color = red>ShaderFeatures()</font>;  
2. 初始化屏幕;    

3. 设置是否支持<font color = red>SRPBatcher</font>;  
4. MSAA配置 
5.  <font color = red>Shader.globalRenderPipeline = "UniversalPipeline"</font>用来检查shader的标签是否为<font color = red>"UniversalPipeline"</font>，如果不是则材质报错为粉红色，如果没有设置则匹配所有管线。  
6. <font color = red>LightMaooing.SetDelegate()</font> //设置一个委托，将光源列表转换为传递给烘焙后端的 LightDataGI 结构的列表。必须通过再次调用 ResetDelegate 来进行重置。  
7. 剩下的则是后处理，默认材质的设置。  

#### Render(ScriptableRenderContext renderContext, Camera[] cameras) 为渲染入口函数
 1. GraphicsSettings 图形设置：颜色空间，是否启用色温（默认启用)。  
 2. SetupPerFrameShaderConstants() 设置shader全局需要的变量   
   首先获取场景设置中的环境光，镜面反射探针，镜面反射颜色，lightmap贴图计算的坐标，镜面CubeMap贴图等，然后通过Shader.SetGlobal传送给每个用到的shader；
 3. SortCameras;对相机进行排序
    ```C#
    void SortCameras(List<Camera> cameras)
        {
            if (cameras.Count > 1)
                cameras.Sort(cameraComparison);
        }
    Comparison<Camera> cameraComparison = (camera1, camera2) => { return (int)camera1.depth - (int)camera2.depth; };
    ```
    可以看到对相机的排序是按照相机的深度来的
  4. 最后便是遍历所有相机。
     ```C#
      if (IsGameCamera(camera))
      {
        RenderCameraStack(renderContext, camera);
      }     
      else
      {
        UpdateVolumeFramework(camera, null);
        RenderSingleCamera(renderContext, camera);
      }
     ```
#### RenderCameraStack 
- 获取相机的**UniversalAdditionalCameraData** 组件。该组件保存了相机的一些设置。
  1. 如果**UniversalAdditionalCameraData** 不为空，则获取它的**scriptableRenderer** 成员变量。
  2. **scriptableRenderer** 会去Asset里根据索引找一个
首先判断了一下**CameraStack** ,如果是OverlayCamera则退出。
- UpdateVolumeFramework()
   URP将后处理集中在Volume中，该方法更新VolumeFramework。核心为调用VolumManager.Updata()。
- InitializeCameraData(baseCamera, baseCameraAdditionalData, !isStackedRendering, out var baseCameraData)
   设置一些相机参数，还有变换矩阵,渲染目标贴图的描述符，都在这里设置了。
#### RenderSingleCamera
> 渲染的核心逻辑初始化灯光，阴影等数据。

 