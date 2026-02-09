\> [!caution] This page contained a drawing which was not converted.   

![Exported image](Exported%20image%2020260109132104-0.png)

Size_t 是宏 = int = 4  
在32位系统中 4x4 = 16 字节
 ![Exported image](Exported%20image%2020260109132106-1.png)  

真的blocks就是指针 每一个指针连接着一个链表
    
流程分析

![Exported image](Exported%20image%2020260109132108-2.png)

1.初始化控制块 （mem偏移到剩余内存的开头）

![Exported image](Exported%20image%2020260109132109-3.png)

1.1.避免野指针 全指向Null Block 方便判断是否块为空
 
1.2 又给头部留了8个字节 （不知道为什么）然后向下取整

![Exported image](Exported%20image%2020260109132111-4.png)  

内存池子最小12byte 最大1GB
 
然后偏移一下 把prev block顶上去 指针变成它的末尾了而已

![Exported image](Exported%20image%2020260109132113-5.png)  

![Exported image](Exported%20image%2020260109132117-6.png)      
 
People Speek