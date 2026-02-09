---
creation_date: 2026-02-05 15:44
type: #Type/Concept
status: #Status/Refactoring
tags: [union, 联合体, 内存复用, C, C++, 类型转换]
aliases: [联合体, 联合, union 拆解]
tech_stack: #Tech/CPP
complexity: ⭐⭐⭐
related_modules: []
---

## 核心摘要

> **union 的本质：多个变量共享同一块内存，大小由最大的成员决定，写入任何一个成员都会覆盖其余所有成员。**

---

## 详细分析

### 底层原理 / 核心规律

```
┌─────────────────────────────┐
│         union 的内存         │  ← 大小 = sizeof(最大成员)
│  ┌────────┐                 │
│  │   i    │  (4 bytes)      │  ← 所有成员从地址 0 开始重叠
│  ├────────┤                 │
│  │   f    │  (4 bytes)      │
│  ├────────┴───────┐         │
│  │       d        │(8 bytes)│  ← 最大 → 决定 union 总尺寸
│  └────────────────┘         │
└─────────────────────────────┘
```

两条核心规律：
1. **所有成员起始地址相同**（都是 union 自身的地址）
2. **写最后，读最后** —— 写入哪个成员，就只能安全地读回哪个成员（读别的是未定义行为）

对比 `struct`：
| | struct | union |
|---|---|---|
| 内存布局 | 成员**依次排列** | 成员**全部重叠** |
| 总大小 | 所有成员之和（+对齐） | 最大成员的大小 |
| 同时访问 | ✅ 各自独立 | ❌ 互斥 |

---

### 具体用法（按难度从低到高）

---

#### 1. 基础声明与使用

**看代码**

```c
union Color {
    int   rgb;          // 整体颜色值
    char  channels[4];  // R, G, B, A 逐通道访问
};

union Color c;
c.rgb = 0xFF0000FF;     // 写整个 int
printf("%02X", c.channels[0]); // 读单个字节
```

**拆解**

```c
union Color {
//    ↑↑↑↑↑  ↑↑↑↑↑↑↑↑↑↑↑↑↑↑
//    类型名   成员列表（共享内存）
    int   rgb;
    char  channels[4];
};
```

**为什么用** —— 同一块数据，需要按"整体"和"拆件"两种方式访问时，避免手动位移计算。

---

#### 2. 带 typedef 的匿名写法（常见于 SDK/引擎）

**看代码**

```c
typedef union {
    float   f;
    int     i;
    unsigned int u;
} FloatBits;

FloatBits val;
val.f = 3.14f;
printf("%08X\n", val.u);  // 直接看 float 的二进制表示
```

**拆解**

```c
typedef union {
// ↑↑↑↑↑↑↑  ↑↑↑↑↑
// 关键词     声明一个匿名 union
    float  f;
    int    i;
    unsigned int u;
} FloatBits;
//  ↑↑↑↑↑↑↑↑
//  typedef 赋的新类型名（用法同 struct）
```

**对比**

```c
// ❌ 不用 typedef —— 每次都要写 union XXX
union FloatBits { float f; int i; };
union FloatBits val;   // C 风格，必须带 union 关键词

// ✅ 用 typedef —— 直接用类型名
typedef union { float f; int i; } FloatBits;
FloatBits val;         // 简洁，C++ 也通用
```

> 注意：C++ 中 `union FloatBits` 可以直接当类型名使用（不需要 typedef），但 C 中不行。

---

#### 3. union 嵌套在 struct 中（最高频场景）

**看代码**

```c
struct Packet {
    int type;           // 决定下面读哪个成员
    union {             // 匿名 union，直接嵌入
        int    intVal;
        float  floatVal;
        char   strVal[16];
    };
};

struct Packet p;
p.type = 1;
p.floatVal = 9.8f;     // 直接访问，不需要 p.data.floatVal
```

**拆解**

```c
struct Packet {
    int type;
    union {             // ← 匿名 union（无标签名、无变量名）
        int    intVal;  //    成员可以直接用 p.intVal 访问
        float  floatVal;
        char   strVal[16];
    };                  // ← 没有变量名，依赖匿名提升
};
```

**对比**

```c
// ❌ 匿名之前 —— 多一层层级
struct Packet {
    int type;
    union { int i; float f; } data;  // 有变量名 data
};
p.data.f = 9.8f;   // 访问时要经过 .data

// ✅ 匿名 union —— 成员提升到外层
struct Packet {
    int type;
    union { int i; float f; };       // 无变量名
};
p.f = 9.8f;        // 直接访问
```

> ⚠️ 匿名 union/struct 是 C11/C++11 标准特性，早期编译器可能不支持。

---

#### 4. 用 union 实现类型双关（Type Punning）

**看代码**

```c
// 把 float 的二进制位直接解读为 int，不经过强制转换
float f = 1.0f;
union { float f; unsigned int u; } pun;
pun.f = f;
unsigned int bits = pun.u;  // 得到 1.0f 的 IEEE 754 位模式
```

**拆解**

```c
union {
    float        f;   // 写入口
    unsigned int u;   // 读出口（不同类型解读同一内存）
} pun;
//  ↑↑↑
//  变量名，声明的同时就是一个实例
```

**为什么用** —— 这是 C 标准（C99 §6.5.2.3）明确允许的"合法 type punning"方式。用 `memcpy` 也可以，但 union 写法更清楚。

> ⚠️ 在 C++ 中，通过 union 做 type punning 严格来说仍是**未定义行为**；C++ 推荐用 `std::bit_cast`（C++20）或 `memcpy`。

---

#### 5. 带标签的 enum + union（Tagged Union / 变体类型）

**看代码**

```c
enum ValueType { TYPE_INT, TYPE_FLOAT, TYPE_STRING };

struct Value {
    enum ValueType type;   // 标签 —— 记住当前活跃成员
    union {
        int    i;
        float  f;
        char   s[32];
    };
};

// 写入
struct Value v = { .type = TYPE_FLOAT, .f = 3.14f };

// 读出（读之前必须判断 type）
if (v.type == TYPE_FLOAT)
    printf("%f\n", v.f);
```

**拆解**

```c
struct Value {
    enum ValueType type;   // ← 标签（tag）：记录哪个成员是有效的
    union {                // ← 数据区：只有一个成员有意义
        int   i;
        float f;
        char  s[32];       // ← 最大 → 决定 union 大小
    };
};
```

**为什么用** —— 这就是 **Tagged Union**，是编译器以外实现"一个变量存多种类型"的标准套路。Rust 的 `enum` 本质就是这个模式。

---

### 快速读法

遇到一个 union，按这三步理解：

```
第一步：看最大的成员 → 知道内存多大
第二步：看哪里在写入 → 知道当前有效数据是哪个成员
第三步：看哪里在读出 → 读出的成员必须和写入的一致（否则是 UB）
```

如果是 `struct` 嵌套 `union`，通常还有一个 `type`/`tag` 字段告诉你当前活跃成员是哪个。

---

### 关联知识

- `[[c和c++]]`
- `[[编译与解释代码]]`
- `[[C++模板使用与实现细节]]`
