# PostProcessPass
## PostProcessPass构造函数
在ForwardRednerer里调用该构造函数，一个叫postprocess，另一个叫FinalPostProcessPass。

构造函数设置一些后处理贴图的格式。
## setup
在Renderer中调用，设置以下内容
-  m_Descriptor 纹理描述符，参数为相机纹理描述符。会把mipmap关闭
-  m_Source  使用哪张图作为源
-  m_Destination 使用哪张图作为后处理的输出图
-  m_Depth 深度图
-  m_InternalLut 色调映射图
-  是否是finalpass
-  是否有finalpass
-  是否需要转换颜色空间
  
## Render
