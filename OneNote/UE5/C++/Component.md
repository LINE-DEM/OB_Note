![Exported image](pic/Exported%20image%2020260113112315-0.png)

都有一个默认的Component

![Exported image](pic/Exported%20image%2020260113112316-1.png)  
![Exported image](pic/Exported%20image%2020260113112318-2.png)

StaticMesh是继承了Uscene 而且多了一个Mesh
 
所以说 这个StaticMesh可以直接当Root 以为父类型的指针可以指向子类实例

![Exported image](pic/Exported%20image%2020260113112323-3.png)

例1：在游戏一开始就爆炸 这个逻辑不能写到构造中 因为不确保周围的Actor初始化完毕

![Exported image](pic/Exported%20image%2020260113112325-4.png)   
不止继承自Actor的类有CDO 组件类也有CDS（默认子对象）

![Exported image](pic/Exported%20image%2020260113112327-5.png)

这是个工厂函数（在UE中 一般不用new来创建对象 因为需要处理很多内部事务 比如注册对象 让引擎知道它存在） 创建了Subobject后 会返回一个Type*
 ![Exported image](pic/Exported%20image%2020260113112329-6.png)  
![Exported image](pic/Exported%20image%2020260113112330-7.png)

这样就暴露给蓝图了