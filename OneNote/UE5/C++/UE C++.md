UHT 是虚幻添加的一个编译过程 生成元数据 （为了UObject实现GC 而不使用智能ptr的引用计数）
   

2.ue重度反射！！ 标准的C++是不知道一个对象有哪些成员变量的 所以你才能在蓝图里看见他们 反射技术知道了所有的成员 所以也能方便的序列化 又因为有序列化 所以实现了状态同步 差量传输变化的属性（只传输改变的属性）
         

Uobject ：是语言层的基类  
Actor：是游戏实体的基类 （唯一的能力是 挂载组件ActorComponent） 甚至没有坐标信息 坐标是在一个scene Component组件中的
   

Pawn和Controller继承自Actor：  
1.pawn：比如战棋游戏中的棋子 pawn提供基础的移动功能 以及 和Controller的交互 移动功能其实实现在配套的Component中（pawn movenment component）  
character：继承自pawn 提供移动 动画 骨骼 状态机 跳跃 （主要实现在Character Movement Component中）
   

2.Controller：记录了控制的pawn 以及具体的控制方式 玩家通过这个类去 控制pawn  
AI Controller：  
Player Controller：很重度 包含网络传输
      

Player Controller：

![Exported image](pic/Exported%20image%2020260113112227-0.png)  
![Exported image](pic/Exported%20image%2020260113112229-1.png)  
![Exported image](pic/Exported%20image%2020260113112230-2.png)  
![Exported image](pic/Exported%20image%2020260113112231-3.png)  
![Exported image](pic/Exported%20image%2020260113112233-4.png)  
![Exported image](pic/Exported%20image%2020260113112235-5.png)  
![Exported image](pic/Exported%20image%2020260113112236-6.png)