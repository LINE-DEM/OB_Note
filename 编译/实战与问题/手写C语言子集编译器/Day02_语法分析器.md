---
creation_date: 2026-01-23 10:00
type: "#Type/Concept"
status: "#Status/Draft"
tags: [编译, 实战, Parser]
aliases: [Day 2 Parser]
tech_stack: "#Tech/CPP"
complexity: ⭐⭐⭐
related_modules: [Day01_词法分析器, Day03_语义分析, 实践_手写Parser]
---

# Day 2：语法分析器实现

## 目标

实现递归下降语法分析器，将Token流转换为AST。

## 任务清单

- [ ] 定义AST节点类型
- [ ] 实现Parser类框架
- [ ] 实现表达式解析（处理优先级）
- [ ] 实现语句解析
- [ ] 实现函数和程序解析
- [ ] 错误恢复机制
- [ ] AST打印（调试用）
- [ ] 编写测试用例

## AST节点设计

```cpp
// 表达式节点
struct Expr { virtual ~Expr() = default; };
struct NumberExpr : Expr { int value; };
struct VarExpr : Expr { std::string name; };
struct BinaryExpr : Expr { Expr* left; Token op; Expr* right; };
struct CallExpr : Expr { std::string callee; std::vector<Expr*> args; };

// 语句节点
struct Stmt { virtual ~Stmt() = default; };
struct DeclStmt : Stmt { std::string name; Expr* init; };
struct AssignStmt : Stmt { std::string name; Expr* value; };
struct IfStmt : Stmt { Expr* cond; Stmt* then_; Stmt* else_; };
struct WhileStmt : Stmt { Expr* cond; Stmt* body; };
struct ReturnStmt : Stmt { Expr* value; };
struct BlockStmt : Stmt { std::vector<Stmt*> stmts; };

// 函数和程序
struct Function { std::string name; std::vector<std::string> params; BlockStmt* body; };
struct Program { std::vector<Function*> functions; };
```

## 关联知识

- [[语法分析]]
- [[递归下降解析器]]
- [[抽象语法树(AST)]]
