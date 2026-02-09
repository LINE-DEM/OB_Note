需要解决 RAM的数据从哪来 以及读取RAM的地址参数从哪来
 
1.新增了寄存器组的读取地址线 输出地址线 专门从rt参数中获取需要存储的数据  
2.通过一次加法计算来获取RAM的存储地址 （因为这样方便对数组操作 ）

![MUX 311 writeEn 30 MernR 31 26 Opcode 2521 rs 2116...](Exported%20image%2020260109133143-0.png)  

这三行是用来初始化 r0为首地址 r1为具体存入的数值 0代表偏移量
 
新增的第二输出一直链接RAM 需要写入时 输出即可
 
sw其实做了做了一次加法操作 然后得到真实地址 然后写入2号输出的值
 
如果对数组操作 第二次就不用再设置首地址为58了

![movi rl 2 sw r Ir0](Exported%20image%2020260109133146-1.png)   
分析rt这个数据段  
因为可以用WriteEn来控制 rt的指令是读还是写
 ![Write A ddr 1561 im 501 funct](Exported%20image%2020260109133148-2.png)   ![sw r 0 r0 lw 0 r](Exported%20image%2020260109133150-3.png)   
lw指令分析

![311 writeEn 30 MernR 31 26 Opcode Din 2521 rs 1561...](Exported%20image%2020260109133154-4.png)  

因为addr1确实有指令 所以输出2号口确实有参数传递到RAM 所以RAW的WR信号为0 不让他写
 
寄存器和RAW必然一个在写一个在读 所以才能这样设计
    ![E e snakeasm movi 0 1 movirl 4 sw rl 0 ro movi rl ...](Exported%20image%2020260109133201-5.png)