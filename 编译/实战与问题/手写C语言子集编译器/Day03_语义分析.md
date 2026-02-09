---
creation_date: 2026-01-23 10:00
type: "#Type/Concept"
status: "#Status/Draft"
tags: [编译, 实战, 语义分析]
aliases: [Day 3 Semantic]
tech_stack: "#Tech/CPP"
complexity: ⭐⭐⭐
related_modules: [Day02_语法分析器, Day04_代码生成, 语义分析]
---

# Day 3：语义分析实现

## 目标

实现符号表和基本的语义检查。

## 任务清单

- [ ] 实现符号表（作用域管理）
- [ ] 检查变量声明和使用
- [ ] 检查函数声明和调用
- [ ] 类型检查（简单版，只有int）
- [ ] 检查return语句
- [ ] 为AST节点标注类型/符号信息
- [ ] 错误报告
- [ ] 编写测试用例

## 实现要点

```cpp
struct Symbol {
    std::string name;
    enum Kind { VAR, FUNC } kind;
    int stackOffset;  // 局部变量的栈偏移
    int paramCount;   // 函数参数数量
};

class SymbolTable {
    std::vector<std::map<std::string, Symbol>> scopes;
public:
    void enterScope();
    void exitScope();
    void define(const Symbol& sym);
    Symbol* lookup(const std::string& name);
};

// 语义分析器
class SemanticAnalyzer {
    SymbolTable symtab;
public:
    void analyze(Program* prog);
private:
    void analyzeFunction(Function* func);
    void analyzeStmt(Stmt* stmt);
    void analyzeExpr(Expr* expr);
};
```

## 检查项

```c
// 错误示例
int foo() {
    x = 5;      // 错误：x未声明
    return bar(); // 错误：bar未定义
}

int main() {
    int x;
    int x;      // 错误：重复声明
}
```

## 关联知识

- [[语义分析]]
- [[符号表设计]]
- [[作用域与名称解析]]
