先说一种最基本的用法 就是ShaderToy上面 用RayMarching ，这种渲染不透明物体的消耗其实并不是特别大  
我们可以每次沿着视线的非常小的步长方式前进进行相交判断，但是我们可以使用“球体跟踪”会更好（在速度方面和精度方面）。实际上，我们一般不是采用小步长前进判断，而是采取我们所知道的最大安全步长前进  
（只是很常用的优化策略 体积云是半透明需要的是内部密度的采样 而不是求相交点）
    ![Exported image](Exported%20image%2020260112171752-0.png)  
![Exported image](Exported%20image%2020260112171754-1.png)

传统渲染（显式渲染）：高效并行处理三角形 通过光栅化去把这些三角形渲染到屏幕 当然这个过程包含很多外部光照等信息 （具体可以自己百度渲染管线）

![Exported image](Exported%20image%2020260112171756-2.png)

其实很多技术都是为了每一帧去尽可能减少需要渲染的三角形数量

![Exported image](Exported%20image%2020260112171800-3.png)

RayMarching（隐式渲染）：不依赖巨量的三角形也能实现复杂细节的表现（只需要两个三角形）  
但是其实区别于常用的后处理（例如模糊 像素化 光晕等效果） 该有的计算量并没有消失 所以由于性能考虑游戏中也没有大量使用
   
![Exported image](Exported%20image%2020260112171805-4.png)

Unity中如何使用

![Exported image](Exported%20image%2020260112171809-5.png)

1.先确定方向计算（在unity中用深度图重建世界坐标）

![Exported image](Exported%20image%2020260112171811-6.png) ![Exported image](Exported%20image%2020260112171812-7.png)

具体看代码到 sampleCloud
 
一旦我们将某些东西建模为SDF函数，我们如何渲染它？这就是**光线追踪(ray marching)算法**的用武之地  
先讲述固体物体的渲染方式  
具体原理：对于每个像素点 发射射线 计算这个像素的颜色值  
光线追踪(raymarching) 中，整个场景是用有符号距离函数(SDF)来定义的。为了找到视线和场景之间的交点，我们从相机开始，一点一点地沿着视线移动一个点。在每一步，我们都会问“这个点在场景表面内吗？”，或者可选地说：“此时SDF是否评估为负数？”。如果确实如此，我们就完成了  
动态展示：  
[Real-time dreamy Cloudscapes with Volumetric Raymarching - Maxime Heckel's Blog](https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/)
 ![Exported image](Exported%20image%2020260112171814-8.png)     

![Exported image](Exported%20image%2020260112171815-9.png) ![](https://graph.microsoft.com/v1.0/users\('2949454224@qq.com'\)/onenote/resources/0-d1886323a4f24251952da4907e3160e5!1-E615644383DC1596!s567b9276f77247418071ca1ea9449898/$value)

这个GetDist 就是相当于这个场的描述

![How to "t ihtercectiüh? 
float RayM4F-ch vec3 ro, vec3 rd) 
marcA;h9 / Cpkere 7 
float do = O. 
for (int 1=0; i<MAX STEPS 
vec3 p = ro+dO*rd 
float dS = GetDist p) 
do dS 
if dS<SURFACE DIST I I 
return do 
dO>MAX DIST)](https://graph.microsoft.com/v1.0/users\('2949454224@qq.com'\)/onenote/resources/0-949785872e244e07a50c8b8904281931!1-E615644383DC1596!s567b9276f77247418071ca1ea9449898/$value)

==SDF:== 用数学方程隐式定义3D几何形状。（体积云渲染是用密度场描述形体 加上SDF可以渲染出各种形状的云）  
例如，满足此等式的任何3D点都位于半径为1个单位且原点为（0,0,0）的球体表面上：

![](https://graph.microsoft.com/v1.0/users\('2949454224@qq.com'\)/onenote/resources/0-fec84bea244446d19ea836add7f8e445!1-E615644383DC1596!s567b9276f77247418071ca1ea9449898/$value)

因为结果f（x，y，z）也是点与球体表面之间的距离，并且它的符号告诉该点是否在球体表面的内部/外部/上，因此该函数也称为符号距离功能（SDF）  
2D动态展示：[Disk - distance 2D (shadertoy.com)](https://www.shadertoy.com/view/3ltSW2)  
3D效果：[Raymarching - Primitives (shadertoy.com)](https://www.shadertoy.com/view/Xds3zN)  
[Rainforest (shadertoy.com)](https://www.shadertoy.com/view/4ttSWf)

[https://www.youtube.com/watch?v=PGtv-dBi2wE&t=139s](https://www.youtube.com/watch?v=PGtv-dBi2wE&t=139s)

![](https://graph.microsoft.com/v1.0/users\('2949454224@qq.com'\)/onenote/resources/0-1bd7e6042d3c4a64b1d7890a2751620b!1-E615644383DC1596!s567b9276f77247418071ca1ea9449898/$value)