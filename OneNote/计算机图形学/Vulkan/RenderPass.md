RenderPass本质是定义一个完整的渲染流程以及使用的所有资源的描述，可以理解为是一份元数据或者占位符的概念其中不包含任何真正的数据。  
RenderPass的资源可能包括 Color、Depth/Stencil、Resolve Attachment和Input Attachment而这些资源被称为Attachment只是资源描述(元数据)定义一些加载/存储操作以及相应资源的格式。
 
RenderPass其实是通过Framebuffer中包含的ImageView拿到真正的数据(牢记RenderPass只是元数据)。
    
作用：提前把这些细节告诉 GPU驱动程序

![Exported image](Exported%20image%2020260112170323-0.png) ![Exported image](Exported%20image%2020260112170325-1.png)

1.基础待机 所有动画回到这里  
2.点击屏幕播放开机动画  
3.按下录制按钮开启聆听动画  
4.录制完毕进入思考 相机视角切换  
5.检索成功  
6.回复的时候三选一播放

验收问题
   

6.网络消息有可能不回复  
7.角色颜色替换

1.判断每次的idx  
2.语音开头的杂音去除