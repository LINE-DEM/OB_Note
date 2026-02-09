---
creation_date: 2026-01-23 10:00
type: "#Type/Concept"
status: "#Status/Draft"
tags: [编译, 实战, 链接, 测试]
aliases: [Day 5 Link and Test]
tech_stack: "#Tech/CPP"
complexity: ⭐⭐⭐
related_modules: [Day04_代码生成, 项目概述, 链接概述]
---

# Day 5：链接与测试

## 目标

将生成的汇编代码汇编、链接为可执行文件，并进行全面测试。

## 任务清单

- [ ] 编写汇编脚本（使用ml64.exe）
- [ ] 添加CRT入口点或自定义入口
- [ ] 链接生成exe
- [ ] 编写测试套件
- [ ] 测试所有语言特性
- [ ] 修复发现的bug
- [ ] 文档总结

## 构建流程

```bash
# 1. 编译（我们的编译器）
mini-cc test.c -o test.asm

# 2. 汇编（MASM）
ml64.exe /c /Fo:test.obj test.asm

# 3. 链接
link.exe /ENTRY:main /SUBSYSTEM:CONSOLE test.obj kernel32.lib

# 或者一步完成
mini-cc test.c -o test.exe
```

## 测试用例

```c
// test_arithmetic.c
int main() {
    int a = 10;
    int b = 3;
    int c = a + b * 2;  // 16
    return c;
}

// test_control.c
int max(int a, int b) {
    if (a > b) {
        return a;
    } else {
        return b;
    }
}

int main() {
    return max(5, 3);  // 5
}

// test_loop.c
int sum(int n) {
    int result = 0;
    int i = 1;
    while (i <= n) {
        result = result + i;
        i = i + 1;
    }
    return result;
}

int main() {
    return sum(10);  // 55
}
```

## 项目总结

完成这个项目后，你应该理解了：
1. 词法分析如何将字符流转换为Token流
2. 语法分析如何构建AST
3. 语义分析如何检查程序正确性
4. 代码生成如何将AST转换为汇编
5. 汇编和链接如何生成可执行文件

## 关联知识

- [[项目概述]]
- [[链接概述]]
- [[汇编器原理]]
