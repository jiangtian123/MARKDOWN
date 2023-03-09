# Attachment Fetch
## 概念
 在移动端的设备上，GPU渲染是分Tile渲染的，一个Tile的输出作为另一个Tile的输入（相同的像素点再计算一次）在引擎层面抽象为**Attachment Fetch**。

这样可以避免Tile写回内存后再将其作为纹理采样带来的高带宽和高内存的消耗。
## 作用
对于需要两个Pass做的事情可以使用。如果 2个 Pass 都是在同一位置的像素进行处理，所以其实可以利用 on-chip Tile Memory 完美解决这个问题。原理是在 G-Buffer Pass 中将数据临时存储到 on-chip memory 中，不写入到 DDR，在 Lighting pass 中由于是计算的同一位置像素，直接使用 on-chip 上的 G-Buffer 数据来计算光照，最后再写入到 tile 中，完成渲染。所以实际上整个渲染过程中，只有一次是真正写入到 DDR 中，中间所有的写入计算都是在 on-chip 上完成，这样就完全利用了 Tiled 架构的硬件特性，与传统多 Pass 直接写入 DDR 的方法相比，可以获得更好的性能，更低的功耗。
## Attachment Fetch在各个API中的实现方式
###  Vulkan
VK有RenderPass和SubPass的概念
- RenderPass
  每个RenderPass可以设置多个Subpass
- SubPass
  每个Subpass都有自己的Color Attachment和Depth Attachment。

  每个SubPass可以设置Input Attachment可以读取同一个RenderPass 内的之前 SubPass 写入的 Color Attachment。

在 Tiled 架构上，每个在 RenderPass 内的 SubPass 只是将数据临时写在 on-chip memory 上，当作为 input 读取时，实际上只是读取之前写在 on-chip 上的数据，这样在 SubPass 的读-写之间没有任何 on-chip 到 GPU DDR 中的传输开销，也就实现了上述的 AF 机制。

在VK需要五个步骤
- RenderPass 设置 SubPass
- Image Attachment 创建时的设置
- Descriptor 配置
- Shader 编写
- 渲染时切换 SubPass

