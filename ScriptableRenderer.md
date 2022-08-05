# ScriptabRenderer
## Excute
执行渲染的函数，调用每个pass的Excute方法。
1. 调用InternalStartRendering执行每个激活的Pass的**OnCameraSetup**方法。
2. 设置一些全局shader变量
   - worldSpaceCameraPos
   - screenParams
   - scaledScreenParams
   - zBufferParams
   - orthoParams
3. 设置时间变量
   - time
   - sintime
   - costime
   - deltatime
   - timeParamters
4. 排序渲染队列，按照pass设定的Event进行排序
5. SetUpLights调用ForwardLights的Setup
6. 对排好序的渲染队列进行分块
## RendererBlocks策略
有三个队列：
- m_BlockEventLimits，长度为A = Const int k_RenderPassBlockCount = 4。
- m_BlockRanges,长度为 B = A + 1。
- m_BlockRangeLengths 长度为 C = B + 1。
### m_BlockEventLimits
存储的渲染块的边界，划分了四个渲染块
| 渲染块名称               | int值 | 对应渲染Event                   |
| :----------------------- | :---- | :------------------------------ |
| BeforeRendering          | 0     | BeforeRenderingPrepasses        |
| MainRenderingOpaque      | 1     | AfterRenderingOpaques           |
| MainRenderingTransparent | 2     | AfterRenderingPostProcessing    |
| AfterRendering           | 3     | (RenderPassEvent)Int32.MaxValue |
每个渲染块存储是该块结束后的渲染Event
### m_BlockRanges
存储激活的RenderPass按照m_BlockEventLimits划分的块的每个块开始的索引
它的填充代码如下:
```C#
int currRangeIndex = 0;
int currRenderPass = 0;
m_BlockRanges[currRangeIndex++] = 0;

// For each block, it finds the first render pass index that has an event
// higher than the block limit.
for (int i = 0; i < m_BlockEventLimits.Length - 1; ++i)
{
    while (currRenderPass < activeRenderPassQueue.Count &&   
    activeRenderPassQueue[currRenderPass].renderPassEvent < m_BlockEventLimits[i])
    currRenderPass++;
    m_BlockRanges[currRangeIndex++] = currRenderPass;
}
m_BlockRanges[currRangeIndex] = activeRenderPassQueue.Count;
```
