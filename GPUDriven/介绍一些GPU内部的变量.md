# 介绍一些GPU内部的变量

如同我们见过的CS算法，线程之间通信的需求很常见，之前我们一直依靠的是**Shared Mem**，但是这需要线程同步。Kepler架构的GPU引入了一种称为“**shuffle**”的新特性，是几种内联函数，允许一个**warp**内的线程之间读取彼此的寄存器而不用通过内存访问和线程同步来共享数据。

这些内联函数可以分为以下几种：

- warp vote - cross warp predicates
  - ballot,all,any
- warp shuffle - cross warp data exchange
  - Indexed(any to any)shuffle,shuffle up,shuffle down,butterfly(xor)shuffle.
- fragment quad swizzle - fragment quad data exchange and arithmetic
  - quadshuffle

除了最后一个，前两个都可以在任何graphics shaders中使用。

## HOW Does Shuffle Help

- 避免使用共享内存，降低了带宽和延迟
- 不使用任何不可用的共享内存
- 不用任何线程同步操作

## 哪里可以用到

相当多的算法需要用到共享内存，如：

- Reductions
  - 计算，最大值，最小值，和。如bloom，depth-of-field，motion blur。（该算法专门讲过）
- List building
  - 各种剔除操作，需要重新对数据构建的，如将一个InstanceCountBuffer中可见的Instance重新放到一个队列中
- Sorting

## Threads,Warps and SMS

对于当前英伟达的显卡来说，一个Warp内的线程数量是32个，为了防止未来这个数量发生改变，我们可以用以下内联函数确定一个warp内的线程数：

```
HLSL:
	NV_WARP_SIZE
	NvGetLaneID()
GLSL:
	gl_WarpSizeNV[or gl_SubGroupSizeARB]
	gl_ThreadInWarpNV[or gl_SubGroupInvocationARB]
```

## Warp Vote -Cross Warp Predicates

warp vote 允许shader访问一个变量的时候，通过广播机制发送给一个warp中的所有线程。

All/Any变体可以将所有线程的计算结果合并成一个bool值，然后广播给一个Warp中的所有线程。

ballot变体向一个warp中的所有线程提供一个32位的掩码，每个线程使用一位。

使用代码：

```
HLSL:
	uint NvAny(int predicate)
	uint NvAll(int predicate)
	uint NvBallot()
GLSL
	bool anyThreadNV(bool predicate) [or bool anyInvocationARB(bool predicate)]
    bool allThreadsNV(bool predicate) [or bool allInvocationsARB(bool predicate)]
    bool allThreadsEqualNV(bool predicate) [or bool allInvocationsEqualARB(bool predicate)]
    uint  ballotThreadNV(bool predicate) [or uint64_t ballotARB(bool predicate)]
```

Sample

```c++
//获取这个32位的掩码 
uint activeThreads = ballotThreadNV(true);
 //LSB最低有效位，MSB最高有效位，还要区分大小端。
 //在大端中lsb指最右边的位
 //在小段中msb指最左边的位
 uint firstLaneId = findLSB(activeThreads);
 uint lastLaneId = findMSB(activeThreads);
 //gl_ThreadInWarpNV 为当前线程在Warp中的ID
 //
 if (firstLaneId == gl_ThreadInWarpNV)
 {
	oFrag = vec4(1, 1, 0, 0);
 }
 else if (lastLaneId == gl_ThreadInWarpNV)
 {
	oFrag = vec4(1, 0, 1, 0);
 }
 else
 {
	oFrag = color;
 }
```

![](D:\Users\Administrator\Desktop\内存测试\ballorSample.jpg)

## Warp Shuffle - Cross Warp Data Exchange

Cross Warp Data Exchange 允许一个Warp中的活跃（有分支的话，有些线程可能会挂起）线程通过以下四种方式交换数据 (indexed, up, down, xor),：

- Indexed Any-to-Any Shuffle

  ```
    HLSL: 
          int NvShfl(int data, uint index, int width = NV_WARP_SIZE)
    GLSL:
          int shuffleNV(int data, uint index, uint width,  [out bool threadIdValid])
  ```
- Shuffle Up and Shuffle Down

  ```
   HLSL:
          int NvShflUp(int data, uint delta, int width = NV_WARP_SIZE)
          int NvShflDown(int data, uint delta, int width = NV_WARP_SIZE)
  
   GLSL:
          int shuffleUpNV(int data, uint delta, uint width, [out bool threadIdValid])
          int shuffleDownNV(int data, uint delta, uint width, [out bool threadIdValid])
  ```
- Butterfly/XOR Shuffle

  ```
   HLSL:
          int NvShflXor(int data, uint laneMask, int width = NV_WARP_SIZE)
  
   GLSL:
          int shuffleXorNV(int data, uint laneMask, uint width, [out bool threadIdValid])
  ```