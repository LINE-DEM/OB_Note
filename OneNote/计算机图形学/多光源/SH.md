SH只需要一个x值就能模拟光照 光照强度公式 i.normal.x是物体空间下的x

![Exported image](Exported%20image%2020260112171322-0.png)

unity中的实现较为复杂 先是unity只计算一次SHA 和 SHB 要注意Gamma空间问题

![Exported image](Exported%20image%2020260112171327-1.png)  
![Exported image](Exported%20image%2020260112171329-2.png)  
![Exported image](Exported%20image%2020260112171331-3.png)

放到BasePass中 也可以SH_ON中
 ![Exported image](Exported%20image%2020260112171334-4.png)  
![Exported image](Exported%20image%2020260112171336-5.png)

分析这个场景和代码 环境光开启但是物体都是黑色  
原因：环境光走的是BasePass 在计算间接光的时候 走的是球协函数
 ![Exported image](Exported%20image%2020260112171337-6.png)

天空盒也是SH 就相当于探针采样计算出SHA SHB