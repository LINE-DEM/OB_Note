GPU粒子的实现原理和ComputeShader GPU Instance 的基础应用。花最少的时间,了解粒子系统的核心奥义  
[https://www.bilibili.com/video/BV19tQhY1E6C/?vd_source=7556e8b0664287072ced69c095b64227](https://www.bilibili.com/video/BV19tQhY1E6C/?vd_source=7556e8b0664287072ced69c095b64227)
   

文字介绍：  
[https://wenku.csdn.net/column/123g6nyh0o](https://wenku.csdn.net/column/123g6nyh0o)
       
**“****工作组****”****（****Work Group****）**:每个工作组由一定数量的线程组成 并行执行
 
**核函数（****Kernel Function**）:这些函数定义了在工作组中的每个线程应该如何执行计算。核函数执行时，会在全局范围内分配线程，并在指定的维度上进行组织。例如，在一个二维图像处理中，可以将核函数应用到每个像素上，线程会被分配到不同的像素点上进行独立计算。
    
**编写核函数：接受的参数（工作组的大小、局部和全局索引）**