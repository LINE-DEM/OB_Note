unit无光照  
透明

![Unity 2017437 cl Personal 64bit CS18unity CS18c PC...](Exported%20image%2020260112170812-0.png)

第一步获取高度值  
lerp出深浅不同的颜色 再lerp出Fresnel效果 浅水区域的贴图A值低一点 再Lerp出不同透明度 三个参数！  
第二步获取Normal值  
获取世界转uv然后控制uv的till和speed 去采样一个法线贴图  
然后可以用不同的参数去再采样一个法线贴图 然后BlendNormals  
第三步获取反射贴图 用这个normal去扰动 采样的uv值  
第四步

[https://zhuanlan.zhihu.com/p/144818695](https://zhuanlan.zhihu.com/p/144818695)
 
[https://zhuanlan.zhihu.com/p/31670275](https://zhuanlan.zhihu.com/p/31670275)