![Exported image](pic/Exported%20image%2020260113112332-0.png)  
![Exported image](pic/Exported%20image%2020260113112334-1.png)  
![Exported image](pic/Exported%20image%2020260113112338-2.png)   
Pawn类特有的绑定函数 （Bind过程）

![Exported image](pic/Exported%20image%2020260113112339-3.png)  

然后这个MoveForward（）每帧都会执行

![Exported image](pic/Exported%20image%2020260113112341-4.png)     

需要添加移动组件

![Exported image](pic/Exported%20image%2020260113112342-5.png)

这个组件 在其他成员变量中控制速度 当你调用传入value为2的时候也没用 （value只是前后判断）

![Exported image](pic/Exported%20image%2020260113112343-6.png)      
加入相机

![Exported image](pic/Exported%20image%2020260113112344-7.png)  

**需求点罗列**  
人物的渲染风格： 写实 卡通  
人物的嘴型：  
1.UE付费插件 无需开发量 效果好 实现快 （用免费的测试1w秒）  
2.公司自研 效果未知  
3.戈图项目免费插件 效果只能用在卡通  
人物头发：  
1.UE头发 无需开发时间 效果好很多很多  
2.Unity头发 需要开发时间 要调整到自然需要较长时间
   

**选择卡通风格**  
效果上：口型够用 头发效果使用块状 渲染效果要求不高 整体和谐  
技术上：  
1.使用Unity  
2.无需使用云渲染 节省Unity 后端和网页前端开发功能和部署调试时间  
3.嘴型直接使用 只需要0.5day替换的时间  
4.动画逻辑 大部分复用戈图项目