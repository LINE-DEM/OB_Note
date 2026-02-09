## 核心摘要

struct 是**内存布局的蓝图**：定义成员的类型和顺序，编译器据此分配一个连续的内存块。声明公式固定，所有变体都是围绕"如何声明"和"如何初始化"的微调。

---

## 详细分析

### 底层原理 / 核心规律

声明和使用的核心公式：

```
声明：struct 标签名 { 成员列表; };
实例：struct 标签名  变量名;        ← C 写法
      标签名        变量名;        ← C++ 可省略 struct
访问：变量名.成员名                  ← 值类型
      指针名->成员名                ← 指针类型
```

内存中各成员**按声明顺序**依次排列，编译器会在成员之间插入 **padding（对齐填充）** 以满足对齐要求。

---

### 具体用法（按难度从低到高排列）

---

#### 1. 基础声明与实例化

**看代码**

```c
struct Point {
    int x;
    int y;
};

struct Point p;   // 声明实例
p.x = 10;
p.y = 20;
```

**拆解**

```c
struct Point { int x; int y; };
↑↑↑↑↑↑ ↑↑↑↑↑
关键字   标签名   ← 定义 struct 类型

struct Point  p;
↑↑↑↑↑↑ ↑↑↑↑↑  ↑
关键字  标签名  变量名  ← C 中必须写 struct 关键字
```

---

#### 2. typedef 简化（消除重复的 struct 关键字）

**看代码**

```c
typedef struct {
    int x;
    int y;
} Point;              // Point 直接就是类型名

Point p = {10, 20};   // 不用写 struct
```

**拆解**

```c
typedef struct { ... } Point;
↑↑↑↑↑↑↑ ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ ↑↑↑↑↑
typedef  struct 体（可匿名） 新类型名
```

**对比**

```c
// 不用 typedef → 每次都要写 struct
struct Point { int x; int y; };
struct Point p;    // ← 麻烦

// 用 typedef → 类型名直接用
typedef struct { int x; int y; } Point;
Point p;           // ← 干净
```

> **规律回归**：和 `[[函数指针]]` 中的 typedef 规律完全一致 —— **把变量名的位置换成新类型名**。struct 里"变量名的位置"就是大括号后面那个名字。

---

#### 3. C vs C++ 的声明差异

**看代码**

```c
// ===== C =====
struct Dog { char name[20]; };
struct Dog d;      // ✅ 必须写 struct
// Dog d;          // ❌ C 中编译报错

// ===== C++ =====
struct Dog { char name[20]; };
Dog d;             // ✅ C++ 中标签名自动成为类型名
struct Dog d2;     // ✅ 写 struct 也行，兼容 C
```

**拆解**

```
C  → "struct 标签名" 才是完整的类型    → struct Dog
C++→  标签名 自身就是类型              → Dog
      struct 可省略，但仍然合法
```

**为什么**：C++ 声明 struct 时会自动把标签名引入当前作用域作为类型名，C 不会。这也是 C++ 中 struct 和 class 本质相同的原因之一。

---

#### 4. 嵌套 struct 与自引用（链表节点）

**看代码**

```c
// 自引用：struct 内部用指针指向自身 ← 链表核心
struct Node {
    int         value;
    struct Node *next;
};

// 嵌套：struct 内包含另一个 struct 的值
struct Rect {
    struct Point topLeft;
    struct Point bottomRight;
};
```

**拆解**

```c
    struct Node *next;
//  ↑↑↑↑↑↑↑↑↑↑↑  ↑↑↑↑
//  指向自身的类型  变量名
//  必须是指针 ↑ 因为 struct Node 的大小在此时还未确定
```

**对比（关键易错点）**

```c
struct Node {
    struct Node  next;   // ❌ 递归值类型 → 大小无穷 → 编译报错
    struct Node *next;   // ✅ 指针大小固定（4/8字节） → OK
};
```

**为什么用**：链表、树、图等递归数据结构的基本模式。自引用必须通过**指针**间接实现。

---

#### 5. 初始化方式对比

**看代码**

```c
struct Config {
    int width;
    int height;
    int fullscreen;
};

// 方式A：按顺序初始化（脆弱 ← 字段顺序改了就 silent bug）
struct Config c1 = {1920, 1080, 1};

// 方式B：designated initializer（C99 / C++20）← 推荐
struct Config c2 = { .width = 1920, .height = 1080, .fullscreen = 1 };

// 方式C：部分初始化（未指定字段自动为 0）
struct Config c3 = { .fullscreen = 0 };   // width=0, height=0
```

**拆解**

```c
{ .width = 1920, .height = 1080 }
  ↑↑↑↑↑↑  ↑↑↑↑   ↑↑↑↑↑↑  ↑↑↑↑
  .字段名  值      .字段名  值
  ↑ 点号是关键标识 → 说明是 designated initializer
```

**对比**

```c
// 不用 designated initializer → 依赖顺序，易断
{1920, 1080, 1}

// 用 designated initializer → 顺序无所谓，字段名明确
{ .width=1920, .height=1080, .fullscreen=1 }
```

> **推荐优先级**：designated initializer > 逐字段赋值 > 按序初始化。实际项目中字段经常增删重排，按序初始化会静默产生 bug。

---

### 快速读法

遇到陌生的 struct 声明，按步骤拆解：

```
1. 有没有 typedef？
   → 有：大括号之后最后一个名字 = 新类型名
   → 没有：struct 后面第一个名字 = 标签名

2. 大括号里是什么？
   → 成员列表（类型 + 名字），就是内存布局的蓝图

3. 大括号外面还有名字吗？
   → 有：那是变量实例（声明+定义合并）

4. 初始化有没有 .字段名？
   → 有：designated initializer（安全）
   → 没有：按声明顺序逐个对应（易错）
```

示例：`typedef struct { int x; int y; } Vec2;`
- `typedef` → 定义新类型
- `struct { int x; int y; }` → 匿名 struct，两个字段
- `Vec2` → 新类型名
- 用法：`Vec2 v = { .x = 1, .y = 2 };`

---

## 关联知识

- `[[前向声明]]` — `struct Foo;` 前向声明，用于解决头文件循环依赖，此时不知道内部字段，只能用指针
- `[[函数指针]]` — typedef 规律相通：变量名位置 → 类型名位置
- `[[c和c++]]` — C 和 C++ 中 struct 声明差异的深层原因
- `[[虚函数]]` — C++ 中 struct 可以有成员函数和虚函数，和 class 本质相同（仅默认访问权限不同）
