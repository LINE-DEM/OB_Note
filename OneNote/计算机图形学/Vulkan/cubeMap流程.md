1.创建6张图片资源  
2.添加布局转换的屏障  
作用：  
（1）将布局从 UNDEFINED 转换到 TRANSFER_DST_OPTIMAL  
（2）等待这个布局转换完毕 才能开启的操作设置为 数据写入imageMemoryBarrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;  
需要设置  
imageMemoryBarrier.srcAccessMask = 0; 因为此时图片刚被创建 无需依赖

![Exported image](Exported%20image%2020260112170420-0.png)  
![Exported image](Exported%20image%2020260112170422-1.png)

（3）.设置好屏障后 用命令直接提交这个屏障 定义了这个VKImage的初始未定义布局--\>写入布局 后续写入操作需要等待VKImage的布局转换完毕 （当前问题就是 有可能布局转换未完成 就开始了写入 有可能是还没有绑定完毕）  
（4）.开始使用StageBuffer 用CommandBuffer提交这次内存拷贝 用VkBufferImageCopy定义如何从一个Buffer拷贝到Image (目前的问题 因为是只有一个Vkimage 一直在设置屏障来转换初始写入布局 等到真的开始写入的时候就有问题) 所以要把所有的图片资源放到cpu的Buffer上 然后一次提交

![Exported image](Exported%20image%2020260112170424-2.png)

3.

![Exported image](Exported%20image%2020260112170426-3.png)

问题1：创建了几个Image
 
正确的是创建了6个Buffer

正确流程分析：  
生成一个Tex  
用Tex生成GenImageCubeMap