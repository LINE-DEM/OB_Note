---
creation_date: 2026-02-04 17:07
type: #Type/Concept
status: #Status/Refactoring
tags: [class, struct, 访问权限, public, private, 继承]
aliases: [class和struct区别, class就是struct, class与struct]
tech_stack: #Tech/CPP
complexity: ⭐⭐
related_modules: []
---

## 核心摘要

C++ 中 class 和 struct **编译后完全相同**，唯一区别是**默认访问权限**：struct 默认 public，class 默认 private。这一条规律可以推出所有相关用法和易错点。

---

## 详细分析

### 底层原理 / 核心规律

```
           默认访问权限    默认继承权限
struct     public         public
class      private        private
           ↑ 唯一的区别就在这里，其余一切相同
```

"其余一切相同"指的是：
- 都可以有成员变量、成员函数、静态成员
- 都可以继承、被继承、有虚函数
- 编译后内存布局完全一致
- sizeof、vtable、构造函数、析构函数 全都一样

> **核心公式**：`class X { 成员; }` 完全等价于 `struct X { private: 成员; }`

---

### 具体用法（按难度从低到高排列）

---

#### 1. 默认访问权限的区别（唯一的语法差异）

**看代码**

```cpp
struct Dog {
    char name[20];     // 默认 public → 外部可直接访问
    int  age;
};

class Cat {
    char name[20];     // 默认 private → 外部无法访问
    int  age;
};

Dog d;
d.name;                // ✅ struct 默认 public
Cat c;
c.name;                // ❌ class 默认 private → 编译报错
```

**拆解**

```cpp
struct Dog { char name[20]; };
↑↑↑↑↑↑                        ← 默认 public（外部可访问）

class  Cat { char name[20]; };
↑↑↑↑↑                         ← 默认 private（外部不可访问）
```

---

#### 2. 显式写访问控制符后，两者 100% 等价

**看代码**

```cpp
// 以下两块代码编译后完全相同，无任何区别：

struct Dog {
public:                // 显式写 public（和 struct 默认一致，可省略）
    char name[20];
    int  age;
};

class Dog {
public:                // 显式写 public → 现在和上面的 struct 版本无区别
    char name[20];
    int  age;
};
```

**对比（习惯约定 vs 编译器现实）**

```cpp
struct Point { int x, y; };              // 习惯用 struct → 简单数据容器
class  Player { public: int x, y; ... }; // 习惯用 class → 有行为的封装对象

// ↑ 用哪个关键字是程序员的风格选择，编译器不在乎
```

---

#### 3. 继承时的默认权限也遵循同一规律（易错点）

**看代码**

```cpp
struct Base {
public:
    void hello() {}
};

struct Child : Base {};        // ✅ 默认 public 继承 → hello() 对外可见
class  Child : Base {};        // ⚠️ 默认 private 继承 → hello() 变成 private！
```

**拆解**

```cpp
class Child : Base {};
↑↑↑↑↑       ↑↑↑↑
默认 private  继承权限也默认 private ← 易漏的点

class Child : public Base {};
              ↑↑↑↑↑↑
              显式写 public → 现在和 struct 继承效果一样
```

**对比（同一继承关系的两种写法）**

```cpp
struct Child : Base {};            // struct → 默认 public 继承 ← 省事
class  Child : public Base {};     // class  → 必须显式写 public 才等价
class  Child : Base {};            // ❌ 漏掉 public → 父类接口静默变 private
```

> **易错点**：用 class 继承时漏掉 `public`，父类的公开接口会**静默变成 private**，外部调用编译报错，但根本原因不明显。

---

#### 4. 前向声明时 class 和 struct 可以互换

**看代码**

```cpp
class  Foo;                     // 前向声明用 class
struct Foo { int x; };          // 定义时用 struct → ✅ 合法，不报错

struct Bar;                     // 前向声明用 struct
class  Bar { public: int x; };  // 定义时用 class → ✅ 也合法
```

**拆解**

```cpp
class  Foo;           // 编译器只记住："Foo 是一个类型名"
struct Foo { ... };   // 定义时换了关键字 → 编译器不区分，不报错
//     ↑↑↑
//     声明和定义用了不同关键字 → C++ 允许
```

> 前一篇 `[[前向声明]]` 也提到了这一点。这进一步证明编译器眼里两者是同一回事。

---

#### 5. 为什么 C++ 要有两个关键字？历史包袱

**不是语言设计选择，是向后兼容的代价。**

```
C 语言          →  只有 struct，无访问控制，无成员函数
    ↓ C++ 要兼容 C
C++ 继承了 struct  →  给 struct 加了 public/private/protected
    ↓ 同时引入 class
C++ 的 class    →  语义暗示"封装对象"，默认 private
C++ 的 struct   →  语义暗示"数据容器"，默认 public（兼容 C 的习惯）
```

**对比**

```
               程序员的语义习惯          编译器看到的
struct Point   "简单数据，不封装"        一个类型，默认 public
class  Player  "封装对象，有行为"        一个类型，默认 private
               ↑ 风格约定               ↑ 编译后无任何区别
```

---

### 快速读法

看到 class 和 struct 混用时，两步判断：

```
第一步：看成员有没有显式写 public / private / protected
   → 有  → class 和 struct 完全无区别，忽略关键字
   → 没有 → 注意默认权限：struct = public，class = private

第二步：看继承写法
   → struct Child : Base       → 默认 public 继承
   → class  Child : Base       → 默认 private 继承（易漏！）
   → class  Child : public Base → 显式 public，效果和 struct 继承一样
```

---

## 关联知识

- `[[Struct]]` — struct 的基础声明语法，C 风格的数据容器用法
- `[[虚函数]]` — 虚函数是 class 最常用的特性，但 struct 同样可以定义虚函数
- `[[前向声明]]` — class 和 struct 的前向声明可以互换，语法/前向声明.md 中有详细拆解
- `[[c和c++]]` — C++ 引入 class 的宏观背景，和 C 的 struct 兼容关系
