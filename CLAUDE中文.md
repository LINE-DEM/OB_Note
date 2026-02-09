# CLAUDE中文.md

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

## 仓库概述

这是一个用于学习 C++ 和虚幻引擎开发的 Obsidian 知识库。该知识库使用中文组织，遵循从底层 C++ 概念到高层虚幻引擎架构模式的结构化学习方法。

**主要语言**：简体中文
**领域**：游戏引擎开发、C++ 编程、虚幻引擎 5

## 知识库结构

### 核心主题区域

1. **c++/** - C++ 基础概念
   - 虚函数和虚表实现
   - 模板机制与实现细节
   - 前向声明
   - C 与 C++ 的区别
   - AActor 与 UActorComponent 架构

2. **CDO/** - 类默认对象 (虚幻引擎)
   - CDO 生命周期与内存模型
   - Pre-main 初始化
   - 原型拷贝机制
   - 构造函数 vs BeginPlay 时机

3. **UE/** - 虚幻引擎特定主题
   - Controller 架构
   - 事件系统
   - 类图解析

4. **编译/** - 编译系统
   - Visual Studio 编译流程
   - UE 编译工作流
   - Build.cs 配置
   - 中间文件和构建产物
   - 编译 vs 解释

5. **设计模式/** - 设计模式
   - 单例和上帝类模式
   - 策略和桥接模式
   - 工厂和建造者模式
   - Playable API 封装综合案例

6. **Templates/** - 文档模板
   - `AI_Refactor_Rules.md` - 将原始笔记重构为结构化文档的标准格式
   - `Architect_Dashboard.md` - 架构仪表板模板
   - `object类.md` - 对象类模板

### 学习路径

该知识库遵循 `学习路径.md` 中记录的系统化学习进度：
- **第一阶段 (第 1-3 周)**：编译系统 - 从 .c 到 .exe
- **第二阶段 (第 4-10 周)**：操作系统基础 - 引导加载程序、进程调度、文件系统
- **第三阶段 (第 11-12 周)**：集成 - 理解 Unity/UE 运行时启动

**知识断层**：该知识库旨在弥合嵌入式系统 (STM32) 与应用层游戏引擎开发之间的知识鸿沟。

## 文档标准

### Obsidian 元数据格式

所有技术笔记都遵循 `Templates/AI_Refactor_Rules.md` 中定义的 YAML 前置元数据结构：

```yaml
---
creation_date: YYYY-MM-DD HH:mm
type: #Type/Concept | #Type/Architecture | #Type/Meeting | #Type/Review
status: #Status/Refactoring
tags: [关键词1, 关键词2, 关键词3]
aliases: [别名1, 别名2]
tech_stack: #Tech/Unity | #Tech/UE5 | #Tech/CPP | #Tech/System
complexity: ⭐⭐⭐ (1-5 星)
related_modules: []
---
```

### 文档结构模板

1. **核心摘要 (Core Summary)** - 一句话精华
2. **详细分析 (Detailed Analysis)**
   - 背景/痛点 (Background/Pain Points)
   - 底层原理 (Underlying Principles)
   - 解决方案/结论 (Solution/Conclusion)
3. **关联知识 (Related Knowledge)** - 使用 `[[概念]]` 语法的双向链接

## 使用此知识库

### 创建或编辑笔记时

1. **始终使用中文**编写内容，除非引用代码/文档
2. **遵循三段式分析结构**：问题 → 原理 → 解决方案
3. **使用技术精确性**：优先使用底层解释（内存地址、汇编、虚表机制），而非高层抽象
4. **添加双向链接**：使用 `[[概念名称]]` 语法链接到相关概念
5. **包含代码示例**：优先使用带有详细注释的 C++ 示例，解释内存行为

### 需要理解的关键概念

- **CDO (类默认对象)**：为每个 UE 类创建的原型实例，用作生成实例的模板
- **虚函数表**：C++ 多态的实现细节，包括构造过程中的 vptr 初始化
- **Pre-Main**：理解 UE 中 main() 执行之前发生的事情
- **原型拷贝**：UE 如何拷贝 CDO 来创建新实例
- **构建系统**：Native CDO (C++) 与 Blueprint CDO 之间的区别

### 常用链接模式

编写笔记时，建立与这些基础概念的连接：
- `[[虚函数]]` - 用于多态相关主题
- `[[CDO]]` - 用于 UE 对象实例化
- `[[前向声明]]` - 用于头文件依赖问题
- `[[C++模板使用与实现细节]]` - 用于模板相关主题
- `[[编译与解释代码]]` - 用于构建系统问题

## 使用中的 Obsidian 插件

- **obsidian-excalidraw-plugin**：用于技术图表和可视化解释
- **dataview**：用于查询和组织笔记
- **templater-obsidian**：用于模板自动化
- **obsidian-git**：用于版本控制
- **terminal**：用于命令行访问
- **obsidian-local-rest-api**：用于外部集成

## 笔记哲学

此知识库强调**架构理解而非表面知识**：
- 专注于"为什么"而非"是什么"
- 从硬件/汇编层面向上追踪执行
- 将嵌入式系统知识 (STM32) 与高层引擎架构连接起来
- 构建编译、链接和运行时初始化的完整心智模型

目标是消除底层编程（寄存器、中断、链接脚本）与高层游戏引擎开发（Unity/UE5 运行时、JIT 编译、虚拟机）之间的知识鸿沟。
