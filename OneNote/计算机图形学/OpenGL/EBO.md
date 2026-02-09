![Exported image](Exported%20image%2020260109182819-0.png)     
![Exported image](Exported%20image%2020260109182820-1.png)

2是使用当前绑定的vbo  
3是把ebo信息加入到pointer

![Exported image](Exported%20image%2020260109182822-2.png)

现象：第二帧Draw的时候会报错  
错误分析：bindBuffer的时候会解绑vao（如果状态机当前有绑定vao的话），Draw的时候需要ebo信息