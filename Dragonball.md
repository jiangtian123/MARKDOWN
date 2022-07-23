# 渲染优化（一）
>真机：IPhone x 处理器 A11  
XCode 版本13.4.1

## 挂机场景（悟空家）
### 性能总览
 挂机界面CPU耗时38ms，GPU耗时22.15ms，帧数在30帧附近浮动。
### 抓帧数据
1. 数据总览：  
   | 名称     | 数据      |
   | :------- | :-------- |
   | DC       | 1062 个   |
   | 顶点数   | 641016 个 |
   | 占用显存 | 316.6 MB  |
   | 纹理占用 | 250.9 MB  |
   | Buffers  | 65.7 MB   |
> 从这个数据来看，最大的问题为DC数量太多有1062个
2. 渲染事件耗时概览(<font color = red>由大到小排列</font>)   
   如图，渲染事件接近1ms的有五个  
   | 渲染事件             | 耗时   | 所属渲染队列    | 该队列总耗时 | 渲染相机               | 渲染相机耗时 |
   | :------------------- | :----- | :-------------- | :----------- | :--------------------- | :----------- |
   | RenderLoopNewBatcher | 6.28ms | DrawOpaque      | 6.28ms       | MapCamera              | 16.39ms      |
   | RenderLoop           | 2.99ms | DrawTransparent | 2.99         | MapCamera              | 16.39ms      |
   | SMAA                 | 1.03ms | PostProcessing  | 5.81ms       | MapCamera              | 16.39ms      |
   | UberPostProcess      | 0.95ms | PostProcessing  | 5.81         | MapCamera              | 16.39ms      |
   | UberPostProcess      | 0.94ms | PostProcessing  | 4.68ms       | UIPostProcessingCamera | 5.28ms       |
> 后处理有两次，一次在MapCamera,一次在UIPostProcessingCamera,主渲染在MapCamera很奇怪（名字起错了？）
3. 场景渲染相机排序(<font color = red>由大到小排列</font>)   
   | 相机名称               | 渲染耗时 |
   | :--------------------- | :------- |
   | MapCamera              | 16.39ms  |
   | UIPostProcessingCamera | 5.28ms   |
   | UICamera               | 1.14ms   |
   | Base_Camera            | 0.74ms   |
4. 场景shader耗时排序（<font color = red>所有使用该shader的DC累加，由高到底，超过1ms的</font>）   
   | shader名称                       | 引用数量（DC） | 总耗时 | 平均单个DC耗时 | Vs耗时比率 | Ps耗时比率 |
   | :------------------------------- | :------------- | :----- | :------------- | :--------- | :--------- |
   | FAE/Grass                        | 647            | 2.71ms | 0.004ms        | 35%        | 65%        |
   | Bloom1                           | 12             | 1.87ms | 0.15ms         | 32%        | 68%        |
   | UberPost                         | 2              | 1.49ms | 0.745ms        | 0.1%       | 99%        |
   | Bloom2                           | 12             | 1.46ms | 0.12ms         | 44%        | 56%        |
   | Suntail_Floage                   | 8              | 1.29ms | 0.16ms         | 1%         | 99%        |
   | Bloom3                           | 12             | 1.10ms | 0.09ms         | 27%        | 73%        |
   | SubPoxlMorphologicalAntialiasing | 1              | 1.05ms | 1.05ms         | 0.1%       | 99.9%      |
   | Blit                             | 4              | 1.02ms | 0.255ms        | 0.1%       | 99.9%      |
> 这些shader是该场景中超过1ms的，引用数量太多说明需要进行合批，Grass shader引用数量有647个DC，草的渲染可以使用GPU实例化，减少DC，SRPBatcher只减少setpass，不减少DC。  
> 从这里依然可以看出多次后处理的问题。  
> 单个DC耗时可以评估一个shader的效率，如果单个耗时太高，则需要针对shader优化。  
> <font color = red>SubPoxlMorphologicalAntialiasing</font>只有一个DC引用但是耗时1.05ms，问题很大。  
> <font color = red>!!!!!!</font> MapCamera才是渲染的3D相机，所有3D物体都在这个相机里渲染了   
### 根据渲染事件查找瓶颈


   
   
