# TAA

## 概述

用信号处理领域的知识解释就是，采样的频率跟不上信号变化的频率。

渲染中的锯齿类型可以分成以下几种：

- 几何锯齿
	几何的定义是连续的，但是光栅化时是离散采样的（判断一个像素在不在三角形内）。表现效果就是几何的边缘有锯齿。
	
	<img src="https://learnopengl-cn.github.io/img/04/11/anti_aliasing_rasterization.png" style="zoom:150%;" />
	
	<img src="https://pic1.zhimg.com/v2-e8fb6df391da080aae6fcb843fe5f76c_r.jpg" style="zoom:80%;" />
	
	如上两图所示。
	
- 纹理采样锯齿
	
	uv和贴图对应不上时，如贴图太大。这种情况就需要开启Mipmap，双线性过滤，三线性过滤，各向异性过滤等。
	
- 着色锯齿
	
	光照函数是连续的，但是我们计算光照时是用一个像素一个像素去计算的，而像素的一些插值得到的属性如法线，世界坐标用来计算光照方程时，就会产生锯齿，表现效果就是高光的闪烁，颜色的跳变等。

那解决锯齿的最有效的手段就是增加采样频率。比如SSAA。但是一帧之内使用多倍于当前屏幕的分辨率来计算消耗太大。TAA既是把SSAA的计算分摊到多帧来实现抗锯齿。
## 静态场景
指相机和物体都不会移动的场景。
###  一，抖动

通过在一个像素内抖动物体，来模拟增加采样点。

![](https://pic3.zhimg.com/80/v2-060ffe489d8bdadfc0229eeaebd85786_720w.webp)

这个抖动的值，可以通过修改投影矩阵来实现

<img src="https://pic4.zhimg.com/80/v2-89270eef419a538de293034aa6ce212f_720w.webp" style="zoom:150%;" />

公式推导如下：
$$
[x,y,z,w]*\begin{matrix}
 n & 0 & 0 & 0\\
 0 & n & 0 & 0 \\
 J_1 & J_2 & n+f & -n*f \\
 0 & 0& 0 & 1 &
\end{matrix} \tag{1}
$$
在NDC空间下的坐标值等于；
$$
X^` = \frac{n}z*x+J_1;Y^` = \frac{n}z*y+J_2
$$
相当于在NDC空间下使当前顶点在XY方向上偏移了 $J_1$，$J_2$。$J_1$，$J_2$的值要小于一个像素。
抖动的偏移量一般要使用低差序列，保证生成的**子采样点**是均匀分布在一个像素内的。
-  为什么抖动是物体，但是这里要说增加了子采样点呢？我的理解如图所示

<img src="D:\MyMakrDown\ReferencePictures/AddSubPixel.jpg" style="zoom:50%;" />

白色框是一个屏幕，蓝色三角形是不抖动时所在的屏幕坐标的位置。注意看（1，3）位置的像素。如果是4XMSAA，则这个像素多四个子采样点，那蓝色三角形就击中了这个像素。同样是（1，3）位置的像素，当蓝色三角偏移一段距离达到橙色三角形的位置后，击中了这个像素中心，与增加子采样点效果类似（之所以说类似是因为，如果是MSAA那（1，3）这个像素的颜色值是1/4的三角形的颜色值，但是如果是抖动覆盖的像素，那这个值就是100%的颜色值）。

![](https://pic4.zhimg.com/80/v2-e1bd0301d05b10d53fbc40439d1cf3eb_720w.webp)


###  二,混合历史帧
理解了抖动造成的效果后，我们需要混合历史帧的结果来实现抗锯齿。比较费的想法是，保存前七帧的结果与当前帧的结果做混合（8XSSAA），但是这样需要很大的内存空间。所以一般的做法是存之前历史帧的累加结果$$P_n = \alpha*C_n + (1-\alpha)*P_(n-1)$$,也就是用前n-1帧的采样结果加上这一帧的结果混合起来保存为新的历史，可以先认为这里的 $$\alpha$$是取一个比较小的值比如0.05。
- $$P_n$$是当前帧输出的颜色值。

- $$C_n$$是进入TAA前计算的当前帧的颜色图。

- $$P_(n-1)$$是上一帧TAA输出的图。

所以一个很简单的静态场景的TAA的伪代码如下



```
//这个代码是神秘海域给的代码
//但是我看MaxWell大佬的代码他在采样CurrentBuffer时使用的uv = UV - Jitter；
float3 currColor = currBuffer.Load(pos);
float3 historyColor = historyBuffer.Load(pos);
return lerp(historyColor, currColor, 0.05f);
```

关于采样当前帧时UnJitter的我的理解：

- 当前帧的颜色输入图需要使用使用Line的采样。

  因为做TAA时是全屏的后处理，采样的uv值大概率是在像素的中心，uv-jitter大概率还是在这个像素内，如果是Point采样，那就没什么作用吧？（不清楚GPU怎么进行四舍五入的）。但是使用Line采样就会进行双线性插值

- 会导致采样CurrentBuffer的值是模糊的

  但是会是图像更加稳定。
	<img src="D:\MyMakrDown\ReferencePictures/NoAA.jpg" style="zoom: 200%;" />
	NOAA
	<img src="D:\MyMakrDown\ReferencePictures/FXAA.jpg" style="zoom: 200%;" />
	FXAA
	<img src="D:\MyMakrDown\ReferencePictures/TAA.jpg" style="zoom: 200%;" />
	TAA

以上图片来自神秘海域的分享。

## 动态场景



