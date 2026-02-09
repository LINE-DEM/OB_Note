# Day 1：词法分析器实现

## 目标

实现一个能识别mini-C语言所有Token的词法分析器。

## 任务清单

- [ ] 定义TokenType枚举
- [ ] 实现Token结构体
- [ ] 实现Lexer类核心逻辑
- [ ] 处理标识符和关键字
- [ ] 处理数字字面量
- [ ] 处理运算符（包括双字符运算符）
- [ ] 跳过空白和注释
- [ ] 错误处理和位置追踪
- [ ] 编写测试用例

## 实现要点

```cpp
// Token类型
enum class TokenType {
    // 关键字
    KW_INT, KW_IF, KW_ELSE, KW_WHILE, KW_RETURN,
    // 标识符和字面量
    IDENTIFIER, NUMBER,
    // 运算符
    PLUS, MINUS, STAR, SLASH,
    ASSIGN, EQ, NE, LT, GT, LE, GE,
    // 分隔符
    LPAREN, RPAREN, LBRACE, RBRACE, COMMA, SEMICOLON,
    // 特殊
    END_OF_FILE, ERROR
};
```

## 测试用例

```c
// test1.c
int main() {
    int x = 42;
    return x;
}

// 期望Token序列:
// KW_INT, IDENTIFIER("main"), LPAREN, RPAREN, LBRACE,
// KW_INT, IDENTIFIER("x"), ASSIGN, NUMBER(42), SEMICOLON,
// KW_RETURN, IDENTIFIER("x"), SEMICOLON,
// RBRACE, EOF
```

## 关联知识

- [[词法分析]]
- [[Token设计]]
- [[实践_手写Lexer]]
