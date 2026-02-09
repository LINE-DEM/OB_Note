1.创建模块 分为Public 和 private  
外部访问这个模块 只能访问Public中的数据  
当修改自定义模块Slash的时候 如果只修改private的部分 不会重新编译其他模块（引用了Slash的模块）

![Exported image](pic/Exported%20image%2020260113112242-0.png)  

Public头文件应​​尽量避免包含其他模块的完整头文件​​，改用前置声明减少编译依赖。

![Exported image](pic/Exported%20image%2020260113112243-1.png)   
Actor中有很多变量 把他们按行为集中为一个一个的Struct

![Exported image](pic/Exported%20image%2020260113112244-2.png)  
![Exported image](pic/Exported%20image%2020260113112245-3.png)

TEXT宏：转为unicode 里面包含了汉字等语言
 
想要打印到屏幕上：

![Exported image](pic/Exported%20image%2020260113112246-4.png)  

FString::Printf 是虚幻引擎中用于格式化字符串的函数，其参数要求与标准 C 的 printf 类似，需要传递 ​​C 风格字符串指针​​（const TCHAR*）。  
FString::Printf 的 %s 要求的是 const TCHAR*，而 Name 是 FString 对象，两者类型不一致。

![Exported image](pic/Exported%20image%2020260113112248-5.png)  
![Exported image](pic/Exported%20image%2020260113112248-6.png)
   
![Exported image](pic/Exported%20image%2020260113112253-7.png)           

蓝图的BeginPlay优先于c++的BeginPlay