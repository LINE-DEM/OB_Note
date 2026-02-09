1.思考磨砂金属  
金属贴图 为1  
但是光滑度为 0.3
 ![Exported image](pic/Exported%20image%2020260113112637-0.png)  

**参数分析**  
**Slab**：物质切片（可以理解为材质层）  
**Diffuse Albedo**：漫反射率

![Exported image](pic/Exported%20image%2020260113112639-1.png)

光滑的金属：密度很大 如果完全镜面反射 那么漫反射强度为0 也就是黑色 这个漫反射率为0 （虽然）
 
**F0**:当光线垂直（以0度角）撞击表面时，该光线被反射（Reflected）为镜面反射光的比率被称为F0  
**F90**:

![Exported image](pic/Exported%20image%2020260113112640-2.png)   
Roughness: 描述漫反射和镜面反射的比值

![Exported image](pic/Exported%20image%2020260113112642-3.png)   
Anisotropy：基于微表面的各项异性（在不同方向上表现的不同特性 比如：不同的反射率 第一个就是在水平方向上）

![Exported image](pic/Exported%20image%2020260113112644-4.png)  

**SSS MFP**: 介质浓度  
SSS MFP Scale：相乘倍数
 ![Exported image](pic/Exported%20image%2020260113112648-5.png)  

黑色就是没啥介质就透明了 可以对RGB分别控制

![Exported image](pic/Exported%20image%2020260113112649-6.png)  

**SSS Phase Anisotropy**: 1向前散射 -1向后散射

![Exported image](pic/Exported%20image%2020260113112650-7.png)   
**Second Roughness**:在基础的pbr 无法表现出 本身是光滑的物体 但是人为做成磨砂 这样在微观层面 会影响漫反射强度

![Exported image](pic/Exported%20image%2020260113112651-8.png)

**Second Roughness Weight**：权重

![Exported image](pic/Exported%20image%2020260113112653-9.png)  
![Exported image](pic/Exported%20image%2020260113112654-10.png)

1.粗糙度  
2.边缘高光的亮度  
3.各向异性的颜色

菲涅尔：

![Exported image](pic/Exported%20image%2020260113112655-11.png)