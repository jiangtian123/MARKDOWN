# TAA注意事项

- 计算MotionVector时，不要引入Jitter。

- 对于镜面反射的东西需要额外注意。

- 透明物体不支持MotionVectort

- 采样Motionvector要用point的方式，采样history要用linear

- 对历史数据进行线性采样可能会带来不相关的数据

- 需要动态的blender来平衡模糊和jitter

- 对于模糊可以添加一个锐化滤波器，center+center*4-up-down-left-right

- 对于历史数据的Clamp，应该要谨慎，因为它使用周边像素取Clamp，如果是边缘部分，会带来其他的颜色

- 对于一些高频变化的地方，更容易出现Ghosting

  - 草地，因为稀稀疏疏的草地中模型的边缘很多
  - 光照变化比较快的地方，如坑坑洼洼的水泥地

  在高频变化的地方，使用**Clamp（historyColor,min,max）**会无效，因为min和max范围很大

  解决办法

  - Step 1

    使用一些标记模型的手段，如模板，上一帧的深度等，神秘海域使用了•0x18 – the two ghosting bits的模板。核心思想就是区分对象。需要保存上一帧的东西。并且作为这一帧的输入

    ```
    uint currStencil = stencilBuffer.Sample(point, uv);
    uint lastStencil = lastStencilBuffer.Sample(point, uvLast);
    blendFactor = (lastStencil & 0x18) == (currStencil & 0x18) ? blendFactor : 1.f;
    ```

    但是需要注意，不要对边缘做这些事情，因为Taa需要将边缘混合其他像素来做模糊。可以对比周边颜色的值，来判断是不是边缘。

  - Step 2 

    鬼影消失了，但是新显示的物体的边缘会特别锐利，神秘海域添加了一个高斯模糊，当Blendfactor = 1的时候

    ```
    blendFactor = (lastStencil & 0x18) == (currStencil & 0x18) ? blendFactor : 1.f;
    
    float3 blurredCurrColor;
    // Gaussian blur currColor with 3x3 neighborhood 
    if (blendFactor == 1.f)
    	return blurredCurrColor;
    
    ```

  - Step 3

    由于history数据是线性采样的，模板数据是point采样的，边缘会出现失配的现象，就会在运动时留下一个像素的鬼影。神秘海域把模板扩张了一个像素来解决这个问题。还有其他手段来解决这个问题，例如使用深度，给边缘标记。使用运动模糊，Inside制作组使用了运动模糊。
  
- 帧间算法还可以解决其他问题

- 下采样和blur对TAA来说不友好

- TAA在更高的帧率下表现的更好，收敛的更快

- Clamp时可以根据Mv收紧包围盒
