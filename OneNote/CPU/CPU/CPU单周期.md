![Exported image](Exported%20image%2020260109132831-0.png)  
![Exported image](Exported%20image%2020260109132834-1.png)

R：OP全为0时是R指令  
前两个R用来计算 有可能放到蓝色的R中  
shamt是左移2位  
funct是具体操作

![Exported image](Exported%20image%2020260109132839-2.png)  
![Exported image](Exported%20image%2020260109132845-3.png)

lw变址寻址 S2+300做为内存地址 取出后的值 放到s1 也就是17号寄存器  
beq 比较 相同的时候 跳转到pc+4+400的地址去取 代码
 ![Exported image](Exported%20image%2020260109132847-4.png)  
![Exported image](Exported%20image%2020260109132850-5.png)

这个寄存器组 两路写入 一路输出
 ![Exported image](Exported%20image%2020260109132857-6.png)   ![Exported image](Exported%20image%2020260109132901-7.png)  
![Exported image](Exported%20image%2020260109132905-8.png)

单周期 哈佛结构：控制器纯组合逻辑 没有时序概念
 ![Exported image](Exported%20image%2020260109132908-9.png)  
![Exported image](Exported%20image%2020260109132912-10.png)  
![Exported image](Exported%20image%2020260109132915-11.png)  
![Exported image](Exported%20image%2020260109132918-12.png)