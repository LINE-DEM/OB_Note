[https://zhuanlan.zhihu.com/p/640672385](https://zhuanlan.zhihu.com/p/640672385)
    
**1.****先理解****TBR****渲染** **与后处理消耗**  
移动端GPU的常见渲染方式Tile-based renderer，既基于图块渲染（TBR）  
因为CPU GPU都用一块内存 GPU的片上缓存大小装不下整个RenderTarget！  
例如我想对图像做一个高斯模糊，就是画一个全屏的三角形，然后按照屏幕空间UV读取之前的RT输出结果。  
这中间有一个切换RT的过程，因为一般情况下RenderTarget不可以既读又写，而切换RT，就必须保证切换RT之前所有Tile都渲染完毕并写回SystemMemory。  
因此这个切换RT的前后就可以视作两个RenderPass，切换的代价是，一写和多读的带宽消耗。  
百人计划  
[https://www.bilibili.com/video/BV1Bb4y167zU?spm_id_from=333.788.videopod.episodes&vd_source=7556e8b0664287072ced69c095b64227&p=2](https://www.bilibili.com/video/BV1Bb4y167zU?spm_id_from=333.788.videopod.episodes&vd_source=7556e8b0664287072ced69c095b64227&p=2)

![Exported image](Exported%20image%2020260109183103-0.png)

**名词解释：**  
1.SystemMemory ：cpu gpu通用内存  
2.On_Chip Memory:存储Tile的颜色 深度 模板 读写非常快  
3.Stall：GPU两次计算有依赖关系 需要串行 产生的等待叫做Stall  
4.FillRate： ROP运行的时钟频率xROP的个数x每个时钟ROP可以处理的像素个数  
5.ROP：Raster Operator Units 光栅处理单元  
6.TBDR：D是延迟渲染

![Exported image](Exported%20image%2020260109183108-1.png)  
![Exported image](Exported%20image%2020260109183110-2.png)  

Main

![Exported image](Exported%20image%2020260109183113-3.png) ![Exported image](Exported%20image%2020260109183118-4.png)

基于反应物：\<link=policy0\>1\</link\>\<link=policy1\>2\</link\>和生成物：\<link=policy2\>3\</link\>，请输出哪些催化剂能引发这一反应

一个匹配match中的数据

![Exported image](Exported%20image%2020260109183120-5.png)  

任务列表  
- [x] 1.中英文替换 化学  
- [ ] 2.化学大模型的按钮 回到主页自动 切换到普通问答 且隐藏  
- [x] 3.化学交互的所有中英文切换  
- [x] 4.调整化学web的UI缩放  
- [ ] 5.预留接口 byte转图片 然后显示出来  
- [x] 6.默认化学选择界面选择第一个  
- [ ] 7.发送UI按钮的状态  
- [x] 8.化学提问 只显示smilse  
- [x] 9.图片通路  
- [x] 10.关闭化学编辑 记得clear  
- [x] 对话框ui位置调整