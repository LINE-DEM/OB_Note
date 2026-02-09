1.释放内存的时候  
1.如果两边有空闲 就合并 或者给初始Pool  
2.如果两边没有空闲 就托管给DSA
   

2.申请内存的时候  
1.直接从初始Pool拿到  
2.从DAS托管的空闲内存Pool中拿到适合大小的 （核心）
         

对DSA算法而言  
1.何时会插入一个Free内存块呢  
1.Free的时候  
2.Free的时候和周围内存块合并了  
3.malloc的时候 给的内存块过大了 多余的需要插入一个Free内存块
 
2.开发人员需要内存的时候 DSA总会给他大于等于所需要的大小  
（这个就是核心）
 
3.为了合并Free空间 所以每个Block需要记录物理位置相邻的Block
 
4.何时会删除一个Free块  
1.malloc的时候  
2.Free的时候 这个块和旁边的Block合并了 删除了旁边这个Block

原则：  
1.立即合并空闲内存  
立即合并适用于实时系统 比如在分配内存的时候不需要 突然使用大量的时间去合并内存 也让时间变的有可预测性  
但是延迟合并内存的DSA 其实很适合一种系统（就是频繁使用相同大小的内存块的系统）
 
2.分割阈值  
最小16字节 因为程序一般不会单独分配一个int 会是一个stuct  
而且把指针也存进去
 
3.高效 可预测性高 的Best Fit分配策略 叫Good-Fit  
按照大小塞入不同的FreeList并且不排序
 
4.不支持内存的扩展  
不支持sbrk()等操作 需要内存只能通过分配  
不依赖硬件的MMU
 
5.有统一的分配策略  
比如到格莱斯李的算法 使用了4种分配策略 最坏的情况无法预测
 
6.STLF不擦除内存  
别的程序能读取到关键信息

实现：segregated fit 实现一个good-fit policy

![Exported image](Exported%20image%2020260109132045-0.png)

64b到128b作为第一层 是一个指针数组  
平均分为4块作为第二层  
右上角的位掩码是用来表示当前块中是否有free内存  
Free Block自己记录着不是来源 而是物理内存上的前一个块和后一个块 （链表）
 
可以理解为每一个块 其实有两种链表  
1.逻辑上用于segregated的List  
2.记录物理地址相邻的块的链表  
细节：

场景分析1：  
初始化大小为1024byte  
所以一开始只有一块free内存 在2的10次方格子中  
现在程序在main的时候第一次new 32个byte  
就会找到这个空闲格子 然后使用其中的32byte  
剩余的992byte 的首地址 被连接到2的9次方格子中

![Exported image](Exported%20image%2020260109132047-1.png)  

场景分析2：  
如果过了一会 程序free了这个32byte的内存1 DSA会去检查逻辑链表的pre块2的footer指针 这个footer指向的是内存块1的前一个逻辑内存块3的首地址 定位了物理内存的相邻块 是为了做合并操作

![Exported image](Exported%20image%2020260109132050-2.png) ![Exported image](Exported%20image%2020260109132051-3.png)

F：是否是free  
T:是否是块中最后一个

32位是4字节  
一共这个Header 是16字节了已经

![Exported image](Exported%20image%2020260109132053-4.png)

优化为位运算

![Exported image](Exported%20image%2020260109132055-5.png)  

可以创建多个内存池子 但是每一个都需要一块Control内存

![Exported image](Exported%20image%2020260109132056-6.png)  

假设创建内存池=204800byte  
SLI = 5  
2级内存分为2的5次方 = 32份  
在f=12 s=13中有很多空闲内存 但是最大的是5887

![Exported image](Exported%20image%2020260109132058-7.png)