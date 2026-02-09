1.作用：主要是用来跨平台 构建 编译 不需要手动配置Windows构建环境 Linix构建环境等  
2.Cmake只是一个构建工具 不包含库的安装和管理 如果用到第三方库 如Glfw等  
3.这时可以引入一个库管理工具 Vcpkg
      

需要准备的只有 在当前目标环境下的c++工具链环境
 
问题：AB需要一起用 但是A工程引用C1.0 B工程引用C1.5

案例分析

![Exported image](Exported%20image%2020260112170112-0.png)

1.规定了Cmake最低版本  
2.项目的名字  
3.在计算机中寻找这些包  
4.捕获src文件下的所有文件  
5.构建可执行文件 名称是一个宏 = Blackhole 然后是所有文件
 ![Exported image](Exported%20image%2020260112170114-1.png)  

6.链接库操作
 ![Exported image](Exported%20image%2020260112170117-2.png)

7.在Build后 添加一个自动化指令 复制文件

实战：  
运行cmake的时候  
1.会先去系统的lib库中寻找 c++的标准库 只需要配置自己的第三方库即可  
2.输出：是一个project
 ![Exported image](Exported%20image%2020260112170120-3.png)  
![Exported image](Exported%20image%2020260112170121-4.png)  

Link就是去链接第三方库的lib 这个lib又引用了其他的第三方库lib

![Exported image](Exported%20image%2020260112170123-5.png)

GUI工具的使用  
1.找到源代码  
2.设置生成的工程目录  
3.设置环境  
4.配置好build的输出路径  
5.点击生成工程  
6.进入工程 点击BuildAll 就生成了头和Lib  
（glm这个数学库 生成的只有头文件）

![Exported image](Exported%20image%2020260112170124-6.png)  
![Exported image](Exported%20image%2020260112170129-7.png)  
![Exported image](Exported%20image%2020260112170130-8.png)