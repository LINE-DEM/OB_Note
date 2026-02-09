\> [!caution] This page contained a drawing which was not converted.   

![Lxmind Panorama IBL A5iZ HDR Shop cmftStudio HDR L...](Exported%20image%2020260112171159-0.png)  

![0 1_HDRCubemap](Exported%20image%2020260112171202-1.png)

1.只是单纯的纹理大小缩放，一直到mip6才能看出区别  
2.调成镜面的时候 unity会自动生成新的mipmap 质量比较高  
根据粗糙度去采样不同的mipmap层级 mipmap等级约低，0就是原始图像，特别清晰的反光。（实现了PBR中的粗糙度概念）  
3.调成漫反射的时候就会生成很粗糙的mipmap

*********反射探针的IBL**********  
half4 color_cubemap = UNITY_SAMPLE_TEXCUBE_LOD(unity_SpecCube0, reflect_dir, mip_level);  
half3 env_color = DecodeHDR(color_cubemap, unity_SpecCube0_HDR);//确保在移动端能拿到HDR信息  
half3 final_color = env_color * ao * _Tint.rgb * _Tint.rgb * _Expose;

准备好数据的IBL

CS3.2  
IBL就是贴图从新计算而已 可以生成镜面cube贴图和漫反射cube贴图（可以用球协SH优化）  
512就够了

*******漫反射IBL************  
float4 uv_ibl = float4(normal_dir, mip_level);
 
half4 color_cubemap = texCUBElod(_CubeMap, uv_ibl);

******镜面反射IBL**********  
float roughness = tex2D(_RoughnessMap, i.uv);  
roughness = saturate(pow(roughness, _RoughnessContrast) * _RoughnessBrightness);  
roughness = lerp(_RoughnessMin, _RoughnessMax, roughness);  
roughness = roughness * (1.7 - 0.7 * roughness);  
float mip_level = roughness * 6.0;
 
half4 color_cubemap = texCUBElod(_CubeMap, float4(reflect_dir, mip_level));

球协SH  
因为漫反射IBL没啥细节 采样一张512贴图太浪费了  
把cubemap原始数据放到生成软件中 就可以离线生成这一堆数据

![KE R R Y Technical Artist Training Tutorial custom...](Exported%20image%2020260112171207-2.png)

float4 normalForSH = float4(normal_dir, 1.0);  
//SHEvalLinearL0L1  
half3 x;  
x.r = dot(custom_SHAr, normalForSH);  
x.g = dot(custom_SHAg, normalForSH);  
x.b = dot(custom_SHAb, normalForSH);
 
//SHEvalLinearL2  
half3 x1, x2;  
// 4 of the quadratic (L2) polynomials  
half4 vB = normalForSH.xyzz * normalForSH.yzzx;  
x1.r = dot(custom_SHBr, vB);  
x1.g = dot(custom_SHBg, vB);  
x1.b = dot(custom_SHBb, vB);
 
// Final (5th) quadratic (L2) polynomial  
half vC = normalForSH.x*normalForSH.x - normalForSH.y*normalForSH.y;  
x2 = custom_SHC.rgb * vC;
 
float3 sh = max(float3(0.0, 0.0, 0.0), (x + x1 + x2));  
sh = pow(sh, 1.0 / 2.2);

有些可以放到顶点函数里

![SubShader Render TypeOpaque Tags LOD 100 Pass Tags...](Exported%20image%2020260112171211-3.png)

unity内置了球协函数

![Skybox 128 20 So 0 Non ghtrt NOLightrnaps](Exported%20image%2020260112171213-4.png)  

自动生成漫反射探针

自动生成镜面反射探针

half3 env_color = ShadeSH9(float4(normal_dir,1.0));

修改后要点击烘培才能更新两个探针

探针组的使用  
1.创建探针组  
2.创建两个点光源 都设置成baked模式  
3.lighting面板更新一下

![Lighting Settings As e Settings lighting Enlighten...](Exported%20image%2020260112171215-5.png) ![Exported image](Exported%20image%2020260112171217-6.png)

附近的小球插值生成新的小球的SH参数

![Exported image](Exported%20image%2020260112171219-7.png)

探针组生成的SH实现的间接光漫反射

unity默认探针生成的SH  
实现的简介光漫反射

采样由IBL生成的Cubemap漫反射

镜面反射IBL

反射探针的IBL

[简述实时渲染中的](https://zhuanlan.zhihu.com/p/108485929) Global Illumination - 知乎 (zhihu.com)￼解释了 ibl采样技术 总结的很好