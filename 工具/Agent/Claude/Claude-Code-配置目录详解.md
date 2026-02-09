# Claude Code 配置目录详解

## 核心摘要
`C:\Users\Admin\.claude` 是 Claude 的本地配置和缓存目录，存储所有用户会话数据、历史记录、项目信息和系统设置。

## 目录结构总览

```
.claude/
├── .credentials.json          # 认证凭证（API 密钥等）
├── settings.json              # 全局设置配置
├── history.jsonl              # 对话历史记录（JSONL 格式）
├── stats-cache.json           # 统计数据缓存
├── cache/                     # 缓存数据目录
├── debug/                     # 调试日志
├── file-history/              # 文件编辑历史
├── ide/                       # IDE 编辑器配置
├── paste-cache/               # 粘贴板缓存
├── plans/                     # 保存的计划和方案
├── projects/                  # 项目配置和状态
├── plugins/                   # 插件数据
├── shell-snapshots/           # Shell 命令快照
├── statsig/                   # 特性开关和 A/B 测试配置
├── telemetry/                 # 遥测数据
└── todos/                     # 待办事项数据
```

## 详细说明

### 📄 配置文件

#### `.credentials.json`（认证凭证）
- **作用**：存储 API 密钥和身份认证信息
- **大小**：447 字节
- **内容**：加密的凭证数据
- **安全性**：包含敏感信息，不应共享或备份到公开仓库
- **更新时间**：2月5日 09:31

#### `settings.json`（全局设置）
```json
{
  "model": "haiku"
}
```
- **作用**：存储用户偏好设置
- **当前配置**：使用 Haiku 模型作为默认 AI 模型
- **可配置项**：模型选择、主题、语言等
- **大小**：22 字节

#### `history.jsonl`（对话历史）
- **格式**：JSONL（每行一个 JSON 对象）
- **行数**：245 条记录
- **大小**：166 KB
- **内容**：所有与 Claude 的对话历史
- **用途**：用于搜索、回顾、上下文恢复
- **更新频率**：每次对话后更新
- **注意**：可能包含敏感对话内容

#### `stats-cache.json`（统计缓存）
- **大小**：6.1 KB
- **作用**：缓存统计数据，提高界面加载速度
- **内容**：Token 使用量、对话数量、模型选择等统计

### 📁 主要目录

#### `cache/`（缓存目录）
- **作用**：存储临时缓存数据
- **主要文件**：`changelog.md`（更新日志，85 KB）
- **用途**：加速数据查询，减少重复计算
- **清理**：通常可以安全清理以释放磁盘空间

#### `projects/`（项目管理）
- **大小**：多个项目目录
- **命名规则**：`C--{路径}`（冒号用 `--` 替代）
- **存储内容**：
  - 项目配置和元数据
  - 工作环境状态
  - 项目特定的上下文
- **例子**：
  - `C--1-HYFStudy-OB-C---c--` → `C:\1.HYFStudy\OB\C++\c++`
  - `C--Users-Admin--craft-agent-workspaces...` → `C:\Users\Admin\.craft-agent\workspaces...`

#### `plans/`（计划和方案）
- **作用**：保存已生成的实现计划
- **格式**：Markdown 文件
- **命名**：使用 UUID-like 名称（如 `hidden-waddling-parnas.md`）
- **文件大小**：5-18 KB
- **用途**：允许用户回顾和恢复之前的计划

#### `shell-snapshots/`（Shell 快照）
- **作用**：记录执行过的 shell 命令和输出快照
- **格式**：`.sh` 脚本文件
- **命名**：`snapshot-bash-{时间戳}-{随机码}.sh`
- **用途**：调试和恢复之前的命令执行上下文
- **文件数量**：80+ 条快照记录
- **大小**：每条 ~342 字节

#### `ide/`（IDE 编辑器配置）
- **作用**：存储 IDE 相关的配置和状态
- **包含**：编辑器设置、UI 状态、快捷键配置等
- **文件数量**：少量配置文件（lock 文件）

#### `file-history/`（文件编辑历史）
- **作用**：追踪文件的修改历史
- **用途**：支持版本控制和撤销功能
- **内容**：文件编辑记录和变更日志

#### `paste-cache/`（粘贴板缓存）
- **作用**：缓存复制粘贴的内容
- **用途**：快速访问最近粘贴的内容

#### `telemetry/`（遥测数据）
- **作用**：收集使用统计和性能数据
- **内容**：用户行为数据、错误日志等
- **用途**：产品改进和分析
- **隐私**：通常是匿名化数据

#### `statsig/`（特性开关）
- **作用**：管理 A/B 测试和特性开关
- **内容**：实验分配、功能启用/禁用配置
- **用途**：灰度发布新功能

#### `plugins/`（插件系统）
- **作用**：存储已安装的插件信息
- **内容**：插件配置、依赖、元数据

#### `todos/`（待办事项）
- **作用**：存储任务清单数据
- **用途**：支持任务管理功能

#### `debug/`（调试信息）
- **作用**：存储调试日志和错误信息
- **用途**：问题诊断和性能分析
- **更新时间**：最近（2月5日 13:25）

## 数据流向

```
用户交互
  ↓
Claude Code 处理请求
  ↓ ┌─────────────────────────────────┐
  ├→ history.jsonl（记录对话）
  ├→ projects/（更新项目状态）
  ├→ cache/（缓存计算结果）
  ├→ shell-snapshots/（记录命令）
  └→ stats-cache.json（更新统计）
  ↓
生成响应，返回给用户
```

## 关键特点

### 🔒 安全性
- 凭证存储在 `.credentials.json`（加密）
- 历史记录可能包含敏感信息
- 建议：不要将 `.claude` 目录提交到版本控制

### 💾 存储优化
- 使用 JSONL 格式：便于增量追加，无需重写整个文件
- 缓存机制：减少重复计算
- 快照记录：支持恢复和审计

### 🎯 项目隔离
- 每个打开的项目有独立的目录
- 项目路径编码为目录名（冒号替换为 `--`）
- 支持多项目并行工作

### 📊 数据分析
- `stats-cache.json`：快速访问统计信息
- `telemetry/`：收集使用数据用于产品改进

## 常见操作

### 查看对话历史
```bash
# 查看最近的对话记录
tail -100 C:\Users\Admin\.claude\history.jsonl

# 搜索特定主题的对话
grep "虚函数" C:\Users\Admin\.claude\history.jsonl
```

### 清理缓存
```bash
# 清理缓存目录（安全操作）
rm -r C:\Users\Admin\.claude\cache\*

# 清理粘贴板缓存
rm -r C:\Users\Admin\.claude\paste-cache\*
```

### 备份重要数据
```bash
# 备份 history.jsonl（保存对话历史）
cp C:\Users\Admin\.claude\history.jsonl history_backup.jsonl

# 备份 plans 目录（保存计划方案）
cp -r C:\Users\Admin\.claude\plans plans_backup
```

## 与 Craft Agent 的关系

Craft Agent（当前会话环境）的配置存储在：
```
C:\Users\Admin\.craft-agent\
```

而 Claude Code 的配置存储在：
```
C:\Users\Admin\.claude\
```

**区别**：
- `.claude/`：Claude Code IDE 的全局配置
- `.craft-agent/`：Craft Agent 工作区配置（项目级别）

## 总结

`.claude` 目录是 Claude 的大脑，存储了：
- ✅ 用户认证信息（`.credentials.json`）
- ✅ 全局配置（`settings.json`）
- ✅ 完整的对话历史（`history.jsonl`）
- ✅ 项目工作状态（`projects/`）
- ✅ 生成的计划方案（`plans/`）
- ✅ 执行的命令快照（`shell-snapshots/`）
- ✅ 性能和统计数据（`stats-cache.json`）

理解这个目录结构可以帮助你：
1. 管理和维护本地数据
2. 备份重要的对话和计划
3. 优化磁盘空间
4. 调试和诊断问题
5. 理解 Claude Code 的工作原理
