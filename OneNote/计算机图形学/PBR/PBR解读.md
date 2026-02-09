\> [!caution] This page contained a drawing which was not converted.   

D = 单根光线的漫反射 = Kd*C*1/Π
 
Kd = 1-Ks  
Ks = (0.1,0.5,0.8)多少比例的光是反射  
C = M材质（0.1，0.5，0.5） 代表 RGB的染色情况
 
先算Ks  
只要Kd+Ks小于1就算PBR
   

S = 镜面反射 =

![Exported image](Exported%20image%2020260109182833-0.png)   
1.F项  
Ks = Fresnel  
F = f0+( 1 - f0 )(1-(h*v))  
F0 = (RGB) 就是某个物质对于RGB三种颜色的高光反射率
 
2.D项  
D项 = NDF 法线分布函数

![Exported image](Exported%20image%2020260109182834-1.png)   
3.G项
 
A =

![Exported image](Exported%20image%2020260109182838-2.png)

实际应用的时候一般会这样算：  
G = A * B （B=上面这个式子的v换为L）
 
这样就描述了两种情况（这是微表面的AO 不是SSAO）  
1.光逃不出来了  
2.眼睛就是看不到
 
只和表面粗糙度有关 N V L都有关

![Exported image](Exported%20image%2020260109182840-3.png)  
![Exported image](Exported%20image%2020260109182844-4.png)   
金属度如何理解：  
先理解如何表示金属上的锈迹斑斑  
最好的方式是金属渲染金属 锈迹也单独渲染  
但是PBR有一种解决方案  
F0大于0.04表示金属  
就是用金属度去描述当前点的F0 的大小
 
特殊情况的处理  
如果金属度为1 现实中就没有漫反射了 全都是高光反射

![Exported image](Exported%20image%2020260109182846-5.png)  

颜色 = 能量*光的颜色*角度*衰减 = (d+s)*Lc*Li*(N*L)

间接光的Diffuse：

![Exported image](Exported%20image%2020260109182848-6.png)  

一个点的半球积分面积：

![Exported image](Exported%20image%2020260109182849-7.png)  
![Exported image](Exported%20image%2020260109182850-8.png)   
公式合成为可操作部分

![Exported image](Exported%20image%2020260109182855-9.png)   
需要把解析几何（完美的数学公式）转变为黎曼和的形式才能在电脑运算

![Exported image](Exported%20image%2020260109182856-10.png)  
![Exported image](Exported%20image%2020260109182857-11.png)

化简

![Exported image](Exported%20image%2020260109182859-12.png)

半球的所有光线累加后需要÷采样数

![Exported image](Exported%20image%2020260109182900-13.png)   
可以预积分 一套天空盒后面的部分都是一样的
   

回顾

![Exported image](Exported%20image%2020260109182902-14.png)  

漫反射比较简单 只需要对L进行积分 但是间接光的高光反射就难一些了

![Exported image](Exported%20image%2020260109182903-15.png)  

Ue4的SplitSum

![Exported image](Exported%20image%2020260109182909-16.png)

前面部分叫做：Prefilted Color  
后面叫：BRDF
 
直接光是能算出F项的  
1.F项  
Ks = Fresnel  
F = f0+( 1 - f0 )(1-(h*v))  
F0 = (RGB) 就是某个物质对于RGB三种颜色的高光反射率  
但是L 是来自整个天空 所以间接光没办法求  
后续会根据物质的粗糙度+F0来算

![Exported image](Exported%20image%2020260109182911-17.png)  

衍生出了BRDF： 当用V这个视角去看法线为N的点 同时粗糙度为R的时候 取出数据去用于渲染 （把F0拆出去 是物体不一样的部分 剩下的部分就是可以与积分了）

![Exported image](Exported%20image%2020260109182912-18.png)

但是要注意 本来的采样点特别光滑的话只需要一个图片即可 但是为了对应粗糙的点的多条采样点 提前把这多个采样点的数据也渲染到图片上了 累积后 变得模糊了 之后根据物体的粗糙度来插值这些颜色 (其中也有重要性采样的关系 只有这个波瓣里面采样 其他的都太小了 影响不大)

![Exported image](Exported%20image%2020260109182913-19.png)

把数据预先写入LUT 最后的形式为Vec2（A，B ）

![Exported image](Exported%20image%2020260109182914-20.png)   
可以理解为前面为光后面是材质

![Exported image](Exported%20image%2020260109182916-21.png)  

![Exported image](Exported%20image%2020260109182917-22.png) ![Exported image](Exported%20image%2020260109182922-23.png)