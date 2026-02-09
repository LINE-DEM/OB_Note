1.const int * 指向const int类型的指针 指针随便指向  
2.int * const 指向int类型的 const指针

![Exported image](Exported%20image%2020260109132305-0.png)  

**修饰指针：**

**修饰函数：**  
表示函数内部没有对变量的写操作  
表现就是能读取任何成员变量 但是这个函数只能调用const修饰过的函数  
void month() const { m_month++; } 错误

**修饰对象：**  
比如func（const Data& data）{data.month();}  
data也只能访问自己的const方法
