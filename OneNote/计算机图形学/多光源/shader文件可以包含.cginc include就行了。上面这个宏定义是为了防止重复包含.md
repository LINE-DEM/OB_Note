![Exported image](Exported%20image%2020260112171256-0.png)  

shader文件可以包含.cginc include就行了。上面这个宏定义是为了防止重复包含

![Exported image](Exported%20image%2020260112171258-1.png)

更新深度信息是我们手动勾选了阴影才多了这个渲染指令  
三个正方体可以合批（要避免和圆穿插 因为圆的点大于300）

衰减计算

![Exported image](Exported%20image%2020260112171301-2.png)  
![Exported image](Exported%20image%2020260112171303-3.png)

虽然现实中光能无限距离 但是性能需要不在光范围里的物体不发生DrawCall

![Exported image](Exported%20image%2020260112171309-4.png)

可以用unity自带的 用的时候要自己定义下POINT  
采样灯光贴图 坐标0--1 和rampMap一样 为了减少指令数

![Exported image](Exported%20image%2020260112171310-5.png)  

unity默认一个方向光 四个像素光 四个顶点光 剩下都是SH

![Exported image](Exported%20image%2020260112171312-6.png)  
![Exported image](Exported%20image%2020260112171314-7.png)  
![Exported image](Exported%20image%2020260112171316-8.png)  
![Exported image](Exported%20image%2020260112171318-9.png)  

顶点光计算