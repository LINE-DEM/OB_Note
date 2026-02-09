[https://blog.csdn.net/xiajun07061225/article/details/7283929/](https://blog.csdn.net/xiajun07061225/article/details/7283929/) (单纯介绍)  
[https://blog.csdn.net/a1047120490/article/details/106755208](https://blog.csdn.net/a1047120490/article/details/106755208) （Unity的Tex系统）  
[https://blog.csdn.net/whl0071/article/details/130596215](https://blog.csdn.net/whl0071/article/details/130596215) （快速实践 创建FBO 并且直接）
   

FBO其实就是帧缓存对象，有时候渲染一次结束，需要保存处理的结果，当作下一次处理的输入时，我们就可以把上一次的处理纹理保存到帧缓存中，给下一个着色器输入即可
 
获取FBO的方式
 ![Exported image](Exported%20image%2020260109182825-0.png)