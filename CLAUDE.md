# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is an Obsidian knowledge vault for studying C++ and Unreal Engine development. The vault is organized in Chinese and follows a structured learning approach from low-level C++ concepts to high-level Unreal Engine architecture patterns.

**Primary Language**: Chinese (Simplified)
**Domain**: Game engine development, C++ programming, Unreal Engine 5

## Vault Structure

### Core Topic Areas

1. **c++/** - Fundamental C++ concepts
   - Virtual functions and vtable implementation
   - Template mechanics and implementation details
   - Forward declarations
   - C vs C++ distinctions
   - AActor and UActorComponent architecture

2. **CDO/** - Class Default Objects (Unreal Engine)
   - CDO lifecycle and memory model
   - Pre-main initialization
   - Archetype copy mechanism
   - Constructor vs BeginPlay timing

3. **UE/** - Unreal Engine specific topics
   - Controller architecture
   - Event system
   - Class diagrams

4. **编译/** - Compilation systems
   - Visual Studio compilation process
   - UE compilation workflow
   - Build.cs configuration
   - Intermediate files and build artifacts
   - Compilation vs interpretation

5. **设计模式/** - Design patterns
   - Singleton and God Object patterns
   - Strategy and Bridge patterns
   - Factory and Builder patterns
   - Comprehensive case study on Playable API encapsulation

6. **Templates/** - Document templates
   - `AI_Refactor_Rules.md` - Standard format for refactoring raw notes into structured documentation
   - `Architect_Dashboard.md` - Architecture dashboard template
   - `object类.md` - Object class template

### Learning Path

The vault follows a systematic learning progression documented in `学习路径.md`:
- **Phase 1 (Weeks 1-3)**: Compilation systems - from .c to .exe
- **Phase 2 (Weeks 4-10)**: Operating system fundamentals - bootloaders, process scheduling, file systems
- **Phase 3 (Weeks 11-12)**: Integration - understanding Unity/UE runtime startup

**Learning Gap**: The vault addresses knowledge gaps between embedded systems (STM32) and application-level game engine development.

## Documentation Standards

### Obsidian Metadata Format

All technical notes follow a YAML frontmatter structure defined in `Templates/AI_Refactor_Rules.md`:

```yaml
---
creation_date: YYYY-MM-DD HH:mm
type: #Type/Concept | #Type/Architecture | #Type/Meeting | #Type/Review
status: #Status/Refactoring
tags: [keyword1, keyword2, keyword3]
aliases: [synonym1, synonym2]
tech_stack: #Tech/Unity | #Tech/UE5 | #Tech/CPP | #Tech/System
complexity: ⭐⭐⭐ (1-5 stars)
related_modules: []
---
```

### Document Structure Template

1. **核心摘要 (Core Summary)** - One-sentence essence
2. **详细分析 (Detailed Analysis)**
   - 背景/痛点 (Background/Pain Points)
   - 底层原理 (Underlying Principles)
   - 解决方案/结论 (Solution/Conclusion)
3. **关联知识 (Related Knowledge)** - Bidirectional links using `[[concept]]` syntax

## Working with This Vault

### When Creating or Editing Notes

1. **Always use Chinese** for content unless quoting code/documentation
2. **Follow the three-part analysis structure**: Problem → Principle → Solution
3. **Use technical precision**: Prefer low-level explanations (memory addresses, assembly, vtable mechanics) over high-level abstractions
4. **Add bidirectional links**: Link to related concepts using `[[Concept Name]]` syntax
5. **Include code examples**: Prefer C++ examples with detailed comments explaining memory behavior

### Key Concepts to Understand

- **CDO (Class Default Object)**: The archetype instance created for each UE class, used as the template for spawning instances
- **Virtual Function Tables**: Implementation details of C++ polymorphism, including vptr initialization during construction
- **Pre-Main**: Understanding what happens before main() executes in UE
- **Archetype Copy**: How UE copies CDOs to create new instances
- **Build System**: The distinction between Native CDO (C++) and Blueprint CDO

### Common Linking Patterns

When writing notes, establish connections to these foundational concepts:
- `[[虚函数]]` for polymorphism topics
- `[[CDO]]` for UE object instantiation
- `[[前向声明]]` for header dependency issues
- `[[C++模板使用与实现细节]]` for template-related topics
- `[[编译与解释代码]]` for build system questions

## Obsidian Plugins in Use

- **obsidian-excalidraw-plugin**: For technical diagrams and visual explanations
- **dataview**: For querying and organizing notes
- **templater-obsidian**: For template automation
- **obsidian-git**: For version control
- **terminal**: For command-line access
- **obsidian-local-rest-api**: For external integrations

## Note-Taking Philosophy

This vault emphasizes **architectural understanding over surface-level knowledge**:
- Focus on "why" over "what"
- Trace execution from hardware/assembly level upward
- Connect embedded systems knowledge (STM32) with high-level engine architecture
- Build complete mental models of compilation, linking, and runtime initialization

The goal is to eliminate knowledge gaps between low-level programming (registers, interrupts, linker scripts) and high-level game engine development (Unity/UE5 runtime, JIT compilation, virtual machines).
