![Exported image](Exported%20image%2020260112165958-0.png)   
在游戏循环中 根据动画信息 更新每个点的坐标 存入内存 然后 doRendering 发到显存中 再用GPU并行处理渲染

渲染概念： 在GPU上有一块内存 800*600 RGB32 现在给他填充上我们想要的数据 这就是渲染  
操作系统窗口 ： 每种操作系统 都有不同的窗口显示函数 例如windows用window 安卓用SurfaceVIew
 
Surface：就是去把渲染结果 输出到跨平台的抽象窗口上
 
句柄：host只有一个编号 传入Device端 会根据这个句柄 找到对应的一块内存  
LogicalDevice:句柄 这个只是用来操作显卡的句柄

![Exported image](Exported%20image%2020260112170000-1.png) ![Exported image](Exported%20image%2020260112170007-2.png) ![Exported image](Exported%20image%2020260112170009-3.png) ![Exported image](Exported%20image%2020260112170011-4.png) ![Exported image](Exported%20image%2020260112170014-5.png) ![Exported image](Exported%20image%2020260112170016-6.png) ![Exported image](Exported%20image%2020260112170019-7.png) ![Exported image](Exported%20image%2020260112170023-8.png)

每一个SecondaryComandBuffer都可以放到一个线程中去做 然后归并到主线程
 
Pipeline:用哪个三角形 哪个着色器 是否启用深度….  
为每个模型做一个SecondaryComandBuffer  
每一个comand都要 指定  
1.需要绘制的物体 也就是VKBuffer 顶点Buffer  
2.Pipeline  
3.RenderPass  
4.Shader
 
最后合成为一个Buffer后 需要提交到渲染队列里

![Exported image](Exported%20image%2020260112170026-9.png)