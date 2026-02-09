---
creation_date: 2026-02-04 17:36
type: #Type/Concept
status: #Status/Refactoring
tags: [const, 常量限定, 指针, 成员函数, C++基础]
aliases: [const限定符, 常量修饰符, const修饰]
tech_stack: #Tech/CPP
complexity: ⭐⭐⭐
related_modules: []
---

## 核心摘要

`const` 的本质规律是一个：**它修饰的是"离它最近的类型"——默认看左边，左边没有东西就看右边。**

---

## 详细分析

### 底层原理 / 核心规律

所有 `const` 的用法都可以用一条规则推出来：

> **`const` 修饰的是它左边紧邻的类型；如果左边没有类型，就修饰右边的类型。**

这条规则适用于变量、指针、引用、函数参数、返回值、成员函数，所有场景都是这一套逻辑。

编译器的作用：在 **编译阶段** 禁止对被 `const` 修饰的对象进行修改，违反会报错。运行阶段不存在 const 的概念，它是纯粹的编译期约束。

---

### 具体用法（按难度从低到高排列）

---

#### 1. 修饰普通变量（基础常量）

**看代码**

```cpp
const int x = 10;
int const y = 20;   // 和上面完全等价
```

**拆解**

```cpp
const int x = 10;
// ↑↑↑↑↑ ↑↑↑
// 修饰对象  变量名
// → x 本身不能再赋值

int const y = 20;   // const 在右边，左边紧邻类型是 int，效果一样
```

**对比**

```cpp
// ✗ 不用 const
int x = 10;
x = 20;          // OK，可以随意修改，容易引入 bug

// ✓ 用 const
const int x = 10;
x = 20;          // ✗ 编译报错：不能赋值给 const 对象
```

**为什么用** —— 标记不应改变的值，让编译器帮你做检查，避免手动错误。

---

#### 2. 修饰指针与指针指向的对象（核心难点）

这是 `const` 最容易搞混的地方，正好用"左边优先"的规则来拆解。

**看代码**

```cpp
const int* p1 = &x;     // 指向 const 的指针
int* const p2 = &x;     // const 指针（指针本身是常量）
const int* const p3 = &x; // 都是 const
```

**拆解**

```cpp
const int* p1 = &x;
// ↑↑↑↑↑ ↑↑↑  ↑↑
// 修饰 int    指针变量
// → const 在 * 左边，修饰的是 int（即指向的对象）
// → p1 能换指向哪里，但不能通过 p1 修改值

int* const p2 = &x;
//    ↑↑↑↑↑ ↑↑
//    const   指针变量
// → const 在 * 右边，修饰的是指针 p2 本身
// → p2 不能换指向，但可以通过 p2 修改值

const int* const p3 = &x;
// ↑↑↑↑↑       ↑↑↑↑↑ ↑↑
// 修饰 int    修饰 p3   指针变量
// → 两个都是 const，指针和值都不能改
```

**对比（列表梳理）**

| 写法 | 指针能否重新指向 | 能否通过指针修改值 |
|------|:-:|:-:|
| `int* p`             | ✓ | ✓ |
| `const int* p`       | ✓ | ✗ |
| `int* const p`       | ✗ | ✓ |
| `const int* const p` | ✗ | ✗ |

**为什么用** —— `const int*` 常用于函数参数，承诺"我不会改你传来的数据"；`int* const` 常用于内嵌硬件寄存器地址映射，地址固定不变。

---

#### 3. 修饰函数参数

**看代码**

```cpp
void printValue(const int* ptr);     // 承诺不会修改 ptr 指向的值
void swap(int& a, int& b);           // 不是 const，因为要修改
void display(const std::string& str); // const 引用，避免拷贝又不修改
```

**拆解**

```cpp
void display(const std::string& str);
//           ↑↑↑↑↑ ↑↑↑↑↑↑↑↑↑↑↑↑  ↑↑↑
//           const   类型           引用
// → const 修饰的是 string 对象本身
// → & 表示引用传递（不拷贝）
// → 组合效果：传入大对象，不拷贝，也不修改
```

**对比**

```cpp
// ✗ 值传递 —— string 会被拷贝一次，开销大
void display(std::string str);

// ✗ 非 const 引用 —— 暴露了修改权限，语义不清
void display(std::string& str);

// ✓ const 引用 —— 零拷贝 + 保护数据
void display(const std::string& str);
```

**为什么用** —— `const &` 是 C++ 中传入"只读大对象"的标准写法，既高效又安全。

---

#### 4. 修饰成员函数（const 方法）

**看代码**

```cpp
class Player {
    int hp;
public:
    int getHP() const;      // const 成员函数
    void setHP(int newHP);  // 非 const，会修改成员
};

int Player::getHP() const {
    return hp;              // 只读成员变量，合法
}
```

**拆解**

```cpp
int Player::getHP() const {
//                   ↑↑↑↑↑
//                   const
// → const 写在参数列表之后
// → 含义：这个函数保证不会修改当前对象（this）的任何成员变量
// → 本质上等价于：const Player* this
```

**对比**

```cpp
// ✗ 不加 const —— 外面无法判断这个函数是否会修改对象
int getHP();

// ✓ 加 const —— 明确声明"只读"，const 对象也能调用
int getHP() const;
```

**为什么用** —— `const` 对象只能调用 `const` 成员函数。写接口时加 `const` 可以明确"这个函数不会有副作用"，也让编译器帮忙检查函数体内是否违反了这个承诺。

---

#### 5. 修饰返回值

**看代码**

```cpp
const int* getData();              // 返回指向 const 的指针
const std::string& getName() const; // 返回 const 引用
```

**拆解**

```cpp
const int* getData();
// ↑↑↑↑↑ ↑↑↑  ↑↑↑↑↑↑↑↑↑↑
// const  int   返回值（指针）
// → 调用者拿到的指针不能用来修改数据
// → 保护了内部数据的封装性

const std::string& getName() const;
// ↑↑↑  ↑↑↑↑↑↑↑↑↑↑↑↑  ↑       ↑↑↑↑↑
// const  返回类型      引用    成员函数的 const
// → 返回内部字符串的引用（不拷贝）
// → 调用者不能修改它
// → 后面的 const 表示这个函数本身也不修改对象
```

**为什么用** —— 返回内部数据的引用或指针时，用 `const` 限制外部修改权限，维持封装。

---

### 快速读法

遇到一个不认识的 `const` 写法时，按以下步骤快速理解：

1. **找 `const` 的位置**，看它在 `*`（或 `&`) 的左边还是右边
2. **左边 → 修饰指向的对象**；**右边 → 修饰指针/引用本身**
3. 如果是成员函数后面的 `const` → 修饰的是 `this`（整个对象）
4. 如果是返回值前面的 `const` → 调用者拿到的东西是只读的

---

## 关联知识

- [[虚函数]] —— 虚函数与 const 成员函数可以同时使用，`const` 虚函数要求子类覆盖时也加 const
- [[C++模板使用与实现细节]] —— 模板参数中也会出现 const 修饰，规则同上
- [[c和c++]] —— C 语言中的 const 语义更受限（C 的 const 变量不能做数组大小，C++ 可以）
