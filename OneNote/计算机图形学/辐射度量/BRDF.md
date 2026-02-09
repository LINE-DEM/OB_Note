BRDF：需要有一个函数去定义光线L（radiance）从某一个方向来 如何分配irradiance到各个方向去 的radiance
 
先算出一个点光源的出射radiance 同时这个也是物体的入射的radiance 会在这个表面dA转换成 Power 然后这个Power会辐射多少 到一个方向去
 
出射方向的radiance = BRDF * 入射方向的radiance造成的irradiance
 ![BRDF The Bidirectional Reflectance Distribution Fu...](Exported%20image%2020260112170639-0.png)  

把所有方向的入射光加起来

![The Refiection Equation 01 dot frp cur Li p Wi cos...](Exported%20image%2020260112170641-1.png)  

渲染方程： 就是加上了自发光而已

![The Rendering Equation Iyo p wo Le p Do Lip Wi wt ...](Exported%20image%2020260112170643-2.png)        
![Vebinar Ptlhdering Equation as Integral Equation R...](Exported%20image%2020260112170648-3.png)   
泰勒展开类似 这就是全局光照 光线追踪方便做k2后面的项

![Ray Tracing L E KE K2E Emission directly From ligh...](Exported%20image%2020260112170650-4.png)  

概率密度函数

![Review Probabilities Continuous Variable and Proba...](Exported%20image%2020260112170652-5.png)  

求解出射光L：  
只计算光源发射来的光

![feN WiW pWi choose N directions wipdf shadep wo Ra...](Exported%20image%2020260112170653-6.png)  

![Randomly choose N directions wipdf Lo 00 For each ...](Exported%20image%2020260112170655-7.png) ![Problem 1 Explosion of rays as bounces go up rays ...](Exported%20image%2020260112170656-8.png)

如果采样次数N==1 这就叫路径追踪

![Microfacet BRDF What kind of microfacets reflect w...](Exported%20image%2020260112170657-9.png) ![Term Formulae Accurate need to consider polarizati...](Exported%20image%2020260112170702-10.png) ![Normal Distribution Function NDF Beckmann NDF Simi...](Exported%20image%2020260112170703-11.png)

在slope空间的高斯 只是为了定义域可以取无限大的同时 为表面的法线不朝下 分母是为了归一化 就是说Slope Space上的法线积分=1

![Normal Distribution Function NDF The Normal Distri...](Exported%20image%2020260112170705-12.png) ![Normal Distribution Function NDF Comparison Beckma...](Exported%20image%2020260112170707-13.png) ![ShadowingMasking Term Or the geometry term G Accou...](Exported%20image%2020260112170709-14.png)
 
![ShadowingMasking Term A commonly used shadowingmas...](Exported%20image%2020260112170710-15.png)

越粗糙的物体损失的能量越多 需要补全多次反射丢失的能量 用这个模型去模拟由于自身微表面遮挡导致的多次弹射后的出射能量（没什么需要实时计算的 有预计算的表 每个材质的预计算图都不一样）

![TLq5yllaConty ApproximationE Results](Exported%20image%2020260112170712-16.png)     

还要考虑颜色的能量损失（没太懂） 用kulla Conty来补充颜色

![The KullaConty Approximation What if the BRDF has ...](Exported%20image%2020260112170717-17.png)

内部多次反射全部加起来 最后要 * 上之前考虑的不带颜色的BRDF上

![The KullaConty Approximation Therefore the proport...](Exported%20image%2020260112170718-18.png)  

![Last Lecture RealTime PhysicallyBased Materials Mi...](Exported%20image%2020260112170720-19.png)