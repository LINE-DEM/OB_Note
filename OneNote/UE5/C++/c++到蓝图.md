\> [!caution] This page contained a drawing which was not converted.   

**1.****c++****变量** **传递到** **event Graph**
 
1.关键是暴露的变量是否需要读写权限（需要这个成员变量为Public/Protected 因为蓝图类是子类）

![Exported image](pic/Exported%20image%2020260113112300-0.png)

编译代码流程 --\>关闭UE --\> 模块编译

![Exported image](pic/Exported%20image%2020260113112301-1.png)

私有变量也可以强行访问

![Exported image](pic/Exported%20image%2020260113112302-2.png)

这样可以打开当前蓝图类所有的**成员变量**

**2.****函数** **传递到** **event Graph**

![Exported image](pic/Exported%20image%2020260113112303-3.png) ![Exported image](pic/Exported%20image%2020260113112307-4.png) ![Exported image](pic/Exported%20image%2020260113112309-5.png)  

区别：是否会修改成员变量的值