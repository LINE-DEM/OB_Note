1.从屏幕射光 蒙特卡洛求每一点的积分  
PDF重要性采样 举例：一个门 在离光源10m的距离 此时虽然用平均采样 只间隔5° 那么射线发出10m都会露掉门

![Exported image](Exported%20image%2020260112171639-0.png)  

2.RSM Photon Mapping  
从光源开始计算
 
优化：因为漫反射是低频信息 所以可以旁边的像素点用一个信息 屏幕空间的优化 （但是法线方向相差很大或者 点空间距离大的是无效插值点）  
问题：没有检测光的遮挡

![Exported image](Exported%20image%2020260112171645-1.png)  

3.LPV 空间划分 缓存SH 稀疏八叉树

4.SVOGI  
优化1：空间划分相机近的越细致

![Exported image](Exported%20image%2020260112171648-2.png)

优化2：五角星是物体 Texture是屏幕采样区域 只需要改UV即可

![Exported image](Exported%20image%2020260112171651-3.png)

要计算三个投影方向的不透明度 采集光线会穿过这个Voxe

![Exported image](Exported%20image%2020260112171654-4.png)