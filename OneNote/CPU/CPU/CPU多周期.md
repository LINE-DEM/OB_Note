\> [!caution] This page contained a drawing which was not converted.   

![Exported image](Exported%20image%2020260109132926-0.png)  
![Exported image](Exported%20image%2020260109132929-1.png)   
分析取指令  
时钟1. pc--\> IR PC+4操作 存到pc

![Exported image](Exported%20image%2020260109132932-2.png)

时钟2：放入OP解析操作 （如果是位移指令 甚至可以一起在T2周期完成）

![Exported image](Exported%20image%2020260109132934-3.png)

时钟3

![Exported image](Exported%20image%2020260109132937-4.png)

周期4 存放结果

![Exported image](Exported%20image%2020260109132944-5.png)  

LW指令执行

![Exported image](Exported%20image%2020260109132950-6.png)  

跳转指令很厉害 因为取指令的时候T2周期已经计算好了地址 所以只需要一个周期

![Exported image](Exported%20image%2020260109132957-7.png)  
![Exported image](Exported%20image%2020260109132959-8.png)