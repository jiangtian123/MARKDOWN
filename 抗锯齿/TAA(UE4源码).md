# TAA(UE4源码)

## 简介
锯齿的类型可以分为
- 采样类型的锯齿
	真实物体表面的颜色是一个二维的连续函数。贴图就是对这个二维函数的采样。Texture Filtering可以看作对原始连续函数重新采样的过程。
	Texture Filtering有两类操作：Magnification filtering和Minification filtering。这两类操作都会导致走样，但是走样的原因是不同的。
	对于Magnification filtering来说，一个texel经过纹理映射应用到多于一个像素点上，走样表现为近摄像机处格子边缘的锯齿。重新采样原始连续函数得要生成更多的样本，也就是说相对于贴图，需要更高的采样率。这时，反走样的目标是恢复原始连续函数中的高频信息，使用的是Reconstruction Filter。所以，类似sinc函数的filter kernel才是最佳的选择。
	而对于Minification filtering来说，贴图中的信息细节太多了，相对于像素来说是高频信息。走样表现为远离摄像机处的大片噪点。这是由于相对于贴图，远离摄像机处像素的采样频率过低，高频信息转换为低频信息导致走样。此时，重新采样原始连续函数的目标是过滤掉高频信息，使用的是Pre Filter，代价是图像变得更模糊了。
- 几何锯齿
	光栅化时采样不足导致的。
	通过增加样本数量尽可能保留高频信息，然后再通过低通滤波器转换到和屏幕分辨率一致。这是最直接有效的方法，但是计算量很大，存储也有较大开销。增加样本数量有很多种方法。人眼对于有固定规律（structure）的图形非常敏感（比如锯齿），而对于偶然出现或形状不太规律（structureless）的形状（又被称为noise）没那么敏感。样本如果都均匀分布，生成的图像上就会呈现有固定规律的走样图像结构。所以，有很多研究都致力于合理分布样本以将走样转换成噪点。
	形态学的方法。深度和法线在几何边缘往往会有突变。构造合适的filter可以帮助识别这些结构，低通滤波器可以将锯齿平滑。通常使用的FXAA算法就属此类。
	基于帧历史的方法。此方法假设上下两帧之间有较明显的连续性，所以可以利用历史帧信息变相增加采样率，以提高当前帧图像的生成质量,也被称为TAA。
- 着色锯齿
	一般是由于对渲染方程的采样不足，因为渲染方程也是一个连续函数，对某些部分(比如法线，高光等)在空间变化较快(高频部分)采样不足也会造成走样，反映在视觉上一般是图像闪烁或者噪点，这类称之为着色走样(Shading Aliasing)，发生在Shading阶段。
### TAA
将SSAA的计算量分摊到多个帧之间。如图所示，任意一帧只计算一个像素点，随着t帧的累积，就有t个采样像素，再混合结果，就相当于做了t个子像素的超采样。当然，每帧中像素需要做一定的抖动(Jitter)，不然不同帧相同位置的像素值就完全一样，也就没有了分摊多帧的意义。通过对像素分摊至多帧的超采样，来提高采样效率，达到缓解锯齿的目的。

![](https://pic3.zhimg.com/v2-47e8f051f2a23d6237f6c716e45d0486_r.jpg)

## UE4 shader
```
// K = Center of the nearest input pixel.
// O = Center of the output pixel.
//
//          |           |
//    0     |     1     |     2
//          |           |
//          |           |
//  --------+-----------+--------
//          |           |
//          | O         |
//    3     |     K     |     5
//          |           |
//          |           |
//  --------+-----------+--------
//          |           |
//          |           |
//    6     |     7     |     8
//          |           |
//
```

o是像素经过抖动后的位置。
k是当前像素的中心点，即采样点。

```
// T = Center of the nearest top left pixel input pixel.
	// O = Center of the output pixel.
	//
	//          | 
	//    T     |     .
	//          | 
	//       O  | 
	//  --------+--------
	//          | 
	//          | 
	//    .     |     .
	//          | 
```

TemporalAASample 是TAA的主要执行逻辑

一开始初始化一些uv，因为使用的cs，所以uv需要计算。

NearestBufferUV = k

NearestTopLeftBufferUV = t

### 处理深度数据
#### 采样
Inside制作组提出了一个采样附近3X3邻域的深度值，然后采用深度值最小的那个（距离最近的）进行重投影采样历史数据，对物体边缘效果更好。

这一步是提前采样邻域的深度然后放进一个共享内存内

注 ：如果开启了**上采样**就不会执行
UE原话 ：Disables scene depth caching for TAA upsample because the extra screen percentage ALU is making things worst.
#### 找最小的深度值
UE4默认是采样了（左上，右上，左下，右下）两个偏移单位的位置即（-2，-2），和自己的深度值比较。然后用距离相机最近的那个位置的UV作为偏移值。
### 计算位移
使用上一步找到的最近深度的uv去采样位移贴图（如果使用的话）来计算位移

```
float2 BackTemp = BackN * OutputViewportSize.xy;

		#if AA_DYNAMIC
		{
			ENCODED_VELOCITY_TYPE EncodedVelocity = SampleVelocityTexture(InputParams.NearestBufferUV + VelocityOffset);
			bool DynamicN = EncodedVelocity.x > 0.0;
			if(DynamicN)
			{
			//uv上的位移距离
				BackN = DecodeVelocityFromTexture(EncodedVelocity).xy;
			}
			BackTemp = BackN * OutputViewportSize.xy;
		}
		#endif
        //位移的距离
		Velocity = sqrt(dot(BackTemp, BackTemp));
```

使用当前的uv -BackN 得到上一帧的uv值

### 处理当前的input颜色值

采样周边3X3的颜色值，main分支只采样五个，依旧是开头那五个。同时转换到YCocg颜色空间

Filter颜色值时，UE4使用了三个权重

- 当前颜色值的HDR权重

```
//YCocg空间使用
float HdrWeightY(float Color, float Exposure) 
{
	return rcp(Color * Exposure + 4.0);
}
//非YCocg
float HdrWeight4(float3 Color, float Exposure) 
{
	return rcp(Luma4(Color) * Exposure + 4.0);
}
float Luma4(float3 Color)
{
	return (Color.g * 2.0) + (Color.r + Color.b);
}
```

- 当前颜色值的空间权重  

```
GET_SCALAR_ARRAY_ELEMENT(PlusWeights, i)
不知道怎么计算的
```

- 如果开启了TONE

```
Filtered.Color = NeighborsColor * rcp(NeighborsFinalWeight);
Filtered.CocRadius = 0;
```

> 没有看到做UnTomep的操作,如果有景深和屏幕空间反射，处理时会有不一样的计算

### 计算用于裁剪历史数据的包围盒

这个过程有三个算法

- BOX_VARIANCE

  这个算法为方差裁剪，有很多讲解。值得注意的是UE4在最后还用上一步计算出的值和方差包围盒做了钳制

  ```
  NeighborMin = m1 - 1.25 * StdDev;
  NeighborMax = m1 + 1.25 * StdDev;
  //IntermediaryResult.Filtered 为上一步Filter的值
  NeighborMin = min( NeighborMin, IntermediaryResult.Filtered );
  NeighborMax = max( NeighborMax, IntermediaryResult.Filtered );
  ```

  

- BOX_SAMPLE_DISTANCE

  这个基于一个半径做的包围盒，在这个半径里的数据才可用，不在的则丢弃

  ```
  // Do color clamping only within a radius.
  	{
          //像素坐标原点
  		float2 PPCo = InputParams.ViewportUV * InputViewSize.xy + TemporalJitterPixels;
          //像素中心坐标
  		float2 PPCk = floor(PPCo) + 0.5;
          //差
  		float2 dKO = PPCo - PPCk;
  		
  		// Sample 4 is is always going to be considered anyway.
  		NeighborMin = Neighbors[4];
  		NeighborMax = Neighbors[4];
  		
  		// Reduce distance threshold as upsacale factor increase to reduce ghosting.
  		float DistthresholdLerp = UpscaleFactor - 1;
  		float DistThreshold = lerp(1.51, 1.3, DistthresholdLerp);
  
  		#if AA_SAMPLES == 9
  			const uint Indexes[9] = kSquareIndexes3x3;
  		#else
  			const uint Indexes[5] = kPlusIndexes3x3;
  		#endif
  
  		UNROLL
  		for( uint i = 0; i < AA_SAMPLES; i++ )
  		{
  			uint NeightborId = Indexes[i];
  			if (NeightborId != 4)
  			{
  				float2 dPP = float2(kOffsets3x3[NeightborId]) - dKO;
  
  				FLATTEN
  				if (dot(dPP, dPP) < (DistThreshold * DistThreshold))
  				{
  					NeighborMin = MinPayload(NeighborMin, Neighbors[NeightborId]);
  					NeighborMax = MaxPayload(NeighborMax, Neighbors[NeightborId]);
  				}
  			}
  		}
  	}
  ```

  

- BOX_MIN_MAX

  没什么说的，就是从所有颜色值中找到最大最小的那个

	### 采样历史数据

AA_BICUBIC是用三次立方采样

同时会把采样的颜色值转换到Ycocg空间

### 判断这个点是不是运动的

直接上代码吧

```
bool Dynamic4;
{
	#if !AA_DYNAMIC
		#error AA_DYNAMIC_ANTIGHOST requires AA_DYNAMIC
	#endif
	// TODO: try a 2x2 for AA_UPSAMPLE
	bool Dynamic1 = SampleVelocityTexture(InputParams.NearestBufferUV, int2( 0, -1)).x > 0;
	bool Dynamic3 = SampleVelocityTexture(InputParams.NearestBufferUV, int2(-1,  0)).x > 0;
	Dynamic4 = SampleVelocityTexture(InputParams.NearestBufferUV).x > 0;
	bool Dynamic5 = SampleVelocityTexture(InputParams.NearestBufferUV, int2( 1,  0)).x > 0;
	bool Dynamic7 = SampleVelocityTexture(InputParams.NearestBufferUV, int2( 0,  1)).x > 0;

	bool Dynamic = Dynamic1 || Dynamic3 || Dynamic4 || Dynamic5 || Dynamic7;、
    //如果附近的点的速度不为0且上一帧这个点的速度也不为0
	IgnoreHistory = IgnoreHistory || (!Dynamic && History.Color.a > 0);
}
```

对于这个像素点来说，当前帧不在移动，但是上一帧的在移动，则需要忽略掉历史数据

### Clamp历史数据

直接上代码吧

```
#if !AA_CLAMP
	return History;
#elif AA_CLIP
	// Clip history, this uses color AABB intersection for tighter fit.
	//float4 TargetColor = 0.5 * ( NeighborMin + NeighborMax );
	float4 TargetColor = Filtered;
	float ClipBlend = HistoryClip( HistoryColor.rgb, TargetColor.rgb, NeighborMin.rgb, NeighborMax.rgb );
	
	//float DistToClamp = saturate(-ClipBlend) / ( saturate(-ClipBlend) + 1 );
	//float DistToClamp = abs( ClipBlend ) / ( 1 - ClipBlend );
	ClipBlend = saturate( ClipBlend );

	HistoryColor = lerp( HistoryColor, TargetColor, ClipBlend );

	#if AA_FORCE_ALPHA_CLAMP
		HistoryColor.a = clamp( HistoryColor.a, NeighborMin.a, NeighborMax.a );
	#endif
	return HistoryColor;

#else //!AA_CLIP
	History.Color = clamp(History.Color, NeighborMin.Color, NeighborMax.Color);
	History.CocRadius = clamp(History.CocRadius, NeighborMin.CocRadius, NeighborMax.CocRadius);
	return History;
#endif
float HistoryClip(float3 History, float3 Filtered, float3 NeighborMin, float3 NeighborMax)
{
#if 0
	float3 Min = min(Filtered, min(NeighborMin, NeighborMax));
	float3 Max = max(Filtered, max(NeighborMin, NeighborMax));	
	float3 Avg2 = Max + Min;
	float3 Dir = Filtered - History;
	float3 Org = History - Avg2 * 0.5;
	float3 Scale = Max - Avg2 * 0.5;
	return saturate(IntersectAABB(Dir, Org, Scale));
#else
	float3 BoxMin = NeighborMin;
	float3 BoxMax = NeighborMax;
	//float3 BoxMin = min( Filtered, NeighborMin );
	//float3 BoxMax = max( Filtered, NeighborMax );

	float3 RayOrigin = History;
	float3 RayDir = Filtered - History;
	RayDir = select(abs( RayDir ) < (1.0/65536.0), (1.0/65536.0), RayDir);
	float3 InvRayDir = rcp( RayDir );

	float3 MinIntersect = (BoxMin - RayOrigin) * InvRayDir;
	float3 MaxIntersect = (BoxMax - RayOrigin) * InvRayDir;
	float3 EnterIntersect = min( MinIntersect, MaxIntersect );
	return max3( EnterIntersect.x, EnterIntersect.y, EnterIntersect.z );
#endif
}
```

### 如果没有使用三次立方采样则锐化

```
// Blend in non-filtered based on the amount of sub-pixel motion.
float AddAliasing = saturate(HistoryBlur) * 0.5;
float LumaContrastFactor = 32.0;
#if AA_YCOCG // TODO: Probably a bug arround here because using Luma4() even with YCOCG=0.
			// 1/4 as bright.
	LumaContrastFactor *= 4.0;
#endif
float LumaContrast = LumaMax - LumaMin;
AddAliasing = saturate(AddAliasing + rcp(1.0 + LumaContrast * LumaContrastFactor));
IntermediaryResult.Filtered.Color = lerp(IntermediaryResult.Filtered.Color, SampleCachedSceneColorTexture(InputParams, int2(0, 0)).Color, AddAliasing);
```



### 计算混合

```
float LumaFiltered = GetSceneColorLuma4(IntermediaryResult.Filtered.Color);
BlendFinal = lerp(BlendFinal, 0.2, saturate(Velocity / 40));
BlendFinal = max( BlendFinal, saturate( 0.01 * LumaHistory / abs( LumaFiltered - LumaHistory ) ) );
if (bCameraCut)
{
	BlendFinal = 1.0;
}
```

根据计算出来的速度来lerp，是屏幕空间（0-h）的

### 混合

如果忽略历史数据就令 History = IntermediaryResult.Filtered;

## 总结

如果有屏幕空间反射和其他景深，需要有特殊处理。鬼影的问题神秘海域的分享是使用模板来标记或者使用上一帧的深度来对比物体。

UE4和Unity都集成了在TAA中上采样的办法
