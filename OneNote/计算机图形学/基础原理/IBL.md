![L H](Exported%20image%2020260112171130-0.png)  
![KE R R Y Technical Artist Training Tutorial texCUB...](Exported%20image%2020260112171132-1.png)  
![orlgln Cubema Cubemap Local Reflection](Exported%20image%2020260112171134-2.png)

![IatlongESSZNZ Panorama IBL HDR Shop cmftStudio HDR...](Exported%20image%2020260112171136-3.png)

HDR BC6H的压缩格式移动端不支持  
IBL  
选择IBL后自动离线计算生成IBL图像  
这个贴图没必要太大 512*512就够了  
实现间接光的镜面反射和漫反射  
SH  
但是SH实现间接光漫反射更快

![Inspector cen Lighting Global Maps Environment Sky...](Exported%20image%2020260112171138-4.png)

上面是一个全局的光照探针（是用SH实现的 传入出一堆数据）

![half3 view_dir normal ize _Wor IdSpaceCameraPos xy...](Exported%20image%2020260112171140-5.png)

IBL高光实现

![Exported image](Exported%20image%2020260112171145-6.png)

IBL漫反射实现

![half4 color_cubemap UNITY reflect_dir mip_level](Exported%20image%2020260112171146-7.png)

IBL用天空盒实现

SH实现漫反射

![half3 env_color ShadeSH9 1 0](Exported%20image%2020260112171147-8.png)  

![normalizelight_dir normal_dir _Distort float3 back...](Exported%20image%2020260112171149-9.png)

玉石材质

![TRANSFER VERTEX TO FRAGMENTo LIGHT ATTENUATIONi](Exported%20image%2020260112171156-10.png)

计算点光源的衰减

IBL就是brdf = diffius + spec + IBL 用的就是CubeMap