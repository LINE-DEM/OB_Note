1.main 调用 Application  
2.

![Exported image](Exported%20image%2020260112170034-0.png)

1.glfwInit（）  
2.Instance::create(true)  
3.  
while (!glfwWindowShouldClose(mWindow)) {  
**glfwPollEvents()**;  
}  
4.需要销毁一切

![Exported image](Exported%20image%2020260112170036-1.png)  

分析Vulkan的初始化  
1.创建VkApplicationInfo  
2.创建扩展 getRequiredExtensions() 把扩展  
3.创建VkInstanceCreateInfo  
4.根据Info创建 VKSDK的Instance  
vkCreateInstance(&instCreateInfo, nullptr, **&mInstance**)  
5.最终内容填充了我们自己创建的**instance**中

Vulkan:是很轻量的API 很多东西是没有的  
1.比如坐标的扩展  
2.窗口的扩展  
3.Layer的扩展  
4.Debug的扩展