\> [!caution] This page contained a drawing which was not converted.   

![Exported image](Exported%20image%2020260109183038-0.png)

Blit之后 这个相机的当前渲染目标就变成了Blit中的dst RT了  
所以错误的做法就是第二行自我拷贝了一次  
同时 因为这个cmd添加的调用时机在AfterPost 所以这个CurrentActive就是后处理阶段生成的临时RT

![Exported image](Exported%20image%2020260109183043-1.png)

![Exported image](Exported%20image%2020260109183044-2.png)

如果最后加了OnRenderImage 就先渲染到临时RT上 最后再渲染到No Name（也就是backBuffer上）

如果开启OnRenderImage  
1.Camera.targetTexture会变化为临时纹理  
2.Camera.activeTexture会变成后缓冲区域  
所以能解释

![Exported image](Exported%20image%2020260109183049-3.png)

一般情况获取相机RT的方式：RenderingData.CameraData.renderer.cameraColorTarget