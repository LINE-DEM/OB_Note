## c和c++

## 使用区别

从编程方式来看，C是一门**过程式语言**，强调通过函数和数据结构来组织代码。而C++是一门**多范式语言**，除了支持过程式编程，还支持面向对象编程、泛型编程，甚至函数式编程的某些特性。

让我用一个实际例子来说明。假设你要管理一个学生信息系统，在C语言中，你可能会这样写：

```c
// C语言风格
struct Student {
    char name[50];
    int age;
    float gpa;
};

void printStudent(struct Student* s) {
    printf("姓名: %s, 年龄: %d, GPA: %.2f\n", s->name, s->age, s->gpa);
}

void updateGPA(struct Student* s, float newGPA) {
    s->gpa = newGPA;
}
```

而在C++中，你可以把数据和操作数据的函数封装在一起，形成一个类：

```cpp
// C++风格
class Student {
private:
    string name;  // 注意这里用的是string类，不是字符数组
    int age;
    float gpa;
    
public:
    // 构造函数：创建对象时自动调用
    Student(string n, int a, float g) : name(n), age(a), gpa(g) {}
    
    // 成员函数：封装了对数据的操作
    void print() {
        cout << "姓名: " << name << ", 年龄: " << age 
             << ", GPA: " << gpa << endl;
    }
    
    void updateGPA(float newGPA) {
        gpa = newGPA;
    }
};
```

看到区别了吗？C++通过类把相关的数据和函数组织在一起，这种封装让代码更容易理解和维护。数据被保护在private区域，只能通过public函数访问，这就是**封装性**的体现。



## 原理区别

从编译原理来看，C++编译器要做的工作比C编译器复杂得多。

**名称修饰（Name Mangling）**：C++支持函数重载，允许同名函数有不同参数。为了实现这一点，**编译器**会对函数名进行修饰，把参数类型信息编码进去。比如`void func(int)` = ?add@@YANNN@Z 和`void func(double)` = ?add@@YAHHH@Z在编译后会有不同的符号名。而C语言的函数名就是它本身，不进行修饰。

**虚函数表（VTable）**：

​	**多态**：简单来说，就是“一种接口，多种实现”。基类定义了一个公共的接口（函数），派生类可以有自己的具体实现。通过基类指针或引用调用这个函数时，程序能够根据指针或引用**实际指向的对象类型**，来调用正确的派生类版本，而不是基类的版本。

​	**虚函数表**：一个**静态的、由编译器自动生成的函数指针数组**。这个表包含了某个类中所有**虚函数**的地址。

​	**虚指针**：一个指向该类虚函数表的**指针**。它会被添加到任何一个包含虚函数（或者从有虚函数的类派生而来）的类的实例（对象）中。

**模板机制**：C++的模板是**编译期**的代码生成机制。当你写一个模板函数或模板类时，编译器会根据你使用的具体类型生成相应的代码。这是一种零开销抽象，生成的代码效率和手写的一样高：

```cpp
// 模板函数：可以处理任何类型
template<typename T>
T max(T a, T b) {
    return (a > b) ? a : b;
}

// 使用时编译器会生成对应版本
int x = max(5, 10);        // 生成int版本
double y = max(3.14, 2.71); // 生成double版本
```



