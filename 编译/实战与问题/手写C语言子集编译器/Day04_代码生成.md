---
creation_date: 2026-01-23 10:00
type: "#Type/Concept"
status: "#Status/Draft"
tags: [编译, 实战, 代码生成, x64]
aliases: [Day 4 CodeGen]
tech_stack: "#Tech/CPP"
complexity: ⭐⭐⭐⭐
related_modules: [Day03_语义分析, Day05_链接与测试, 代码生成]
---

# Day 4：代码生成实现

## 目标

生成x64汇编代码（MASM语法）。

## 任务清单

- [ ] 设计栈帧布局
- [ ] 实现表达式代码生成
- [ ] 实现语句代码生成
- [ ] 实现函数代码生成
- [ ] 处理函数调用（Windows x64调用约定）
- [ ] 生成完整的汇编文件
- [ ] 测试生成的汇编能否汇编运行

## 栈帧布局

```
// Windows x64 调用约定
// 参数: RCX, RDX, R8, R9, 栈
// 返回值: RAX
// 调用者保存: RAX, RCX, RDX, R8-R11
// 被调用者保存: RBX, RBP, RDI, RSI, R12-R15

// 栈帧布局
高地址
    ┌─────────────────┐
    │   参数5+（如有）  │
    ├─────────────────┤
    │   Shadow Space  │  ← 32字节，调用者分配
    ├─────────────────┤
    │   返回地址       │
    ├─────────────────┤
    │   保存的RBP     │  ← RBP指向这里
    ├─────────────────┤
    │   局部变量       │
    ├─────────────────┤
    │   临时空间       │  ← RSP指向这里
    └─────────────────┘
低地址
```

## 代码生成示例

```cpp
// 源代码
int add(int a, int b) {
    return a + b;
}

// 生成的汇编（简化）
add PROC
    push rbp
    mov rbp, rsp
    sub rsp, 16           ; 局部变量空间

    mov [rbp-4], ecx      ; 保存参数a
    mov [rbp-8], edx      ; 保存参数b

    mov eax, [rbp-4]      ; 加载a
    add eax, [rbp-8]      ; a + b

    mov rsp, rbp
    pop rbp
    ret
add ENDP
```

## 关联知识

- [[代码生成]]
- [[x64汇编速查]]
- [[寄存器分配]]
