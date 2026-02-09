# AI Agent、MCP、Obsidian 交互流程详解

## 概述

这是一个三层架构，每层都有明确的职责：

| 角色      | 组件                                   | 职责                     |
| ------- | ------------------------------------ | ---------------------- |
| **老板**  | AI 客户端 (Craft Agent / Cherry Studio) | 理解用户意图，决定调用什么工具        |
| **翻译官** | MCP Server (obsidian-mcp-server)     | 把 MCP 协议转换成目标服务的 API   |
| **服务员** | Local REST API 插件                    | 接收 HTTP 请求，操作 Obsidian |
| **厨房**  | Obsidian + 笔记文件                      | 实际存储数据的地方              |

## 交互流程

以"搜索笔记"为例：

```
1. 用户 → AI：「帮我搜索关于指针的笔记」

2. AI 理解意图，决定调用 search_notes 工具

3. AI → MCP Server：发送 MCP 协议消息
   {
     "method": "tools/call",
     "params": {
       "name": "search_notes",
       "arguments": { "query": "指针" }
     }
   }

4. MCP Server → Local REST API：转换为 HTTP 请求
   POST https://127.0.0.1:27124/search/
   Headers: Authorization: Bearer <API_KEY>
   Body: { "query": "指针" }

5. Local REST API：验证 API Key，遍历笔记文件，执行搜索

6. 结果原路返回：REST API → MCP Server → AI → 用户
```

## 配置项详解

### OBSIDIAN_API_KEY
- **是什么**：一个密码/令牌
- **谁用它**：MCP Server 用它向 Local REST API 证明身份
- **类比**：员工工牌，服务员看到工牌才接受指令

### OBSIDIAN_BASE_URL
- **是什么**：Local REST API 的地址
- **格式**：`https://127.0.0.1:27124`
  - `127.0.0.1` = 本机
  - `27124` = 端口号
  - `https` = 加密通信

### OBSIDIAN_VERIFY_SSL
- **是什么**：是否验证 SSL 证书
- **为什么设 false**：Local REST API 用自签名证书，不是正规 CA 颁发的
- **类比**：「我认识这个人，不需要看身份证」

### OBSIDIAN_VAULT_PATH
- **是什么**：笔记库的文件系统路径
- **有些 MCP Server 需要**，有些不需要（取决于实现）

## 通用规律（举一反三）

**MCP Server 的本质 = 协议转换器**

```
AI 说的语言（MCP 协议）  ←→  目标服务说的语言（REST/SQL/GraphQL/...）
```

### 任何 MCP 集成都是这个模式

| 连接什么 | MCP Server | 需要配置什么 |
|----------|------------|--------------|
| Obsidian | obsidian-mcp-server | API Key, URL |
| GitHub | github-mcp-server | Personal Access Token |
| Linear | linear-mcp | OAuth |
| 数据库 | postgres-mcp | 连接字符串 |

### 为什么需要 MCP 这一层？

**没有 MCP**：AI 需要学习每个服务的 API（Obsidian API、GitHub API、Linear API...）

**有了 MCP**：
- AI 只学一种协议（MCP）
- 新服务只需写一个 MCP Server
- 解耦 = 灵活 = 可扩展

## 形象比喻

把整个系统想象成一个国际餐厅：

- **客人（用户）**：用中文点菜
- **老板（AI）**：理解客人需求，用「通用指令语言」（MCP）下单
- **翻译官（MCP Server）**：把指令翻译成厨房能懂的语言
- **服务员（REST API）**：验证身份，传达给厨房
- **厨房（Obsidian）**：实际做菜（读写笔记）

每一层都有自己的「语言」和「认证方式」，但最终目的是：让客人能吃到想吃的菜。

## 两个 API Key 的区别

你可能注意到有两个 Key：

| Key | 在哪里设置 | 用途 |
|-----|-----------|------|
| `OBSIDIAN_API_KEY` (环境变量) | MCP Server 配置 | MCP Server 访问 REST API 时用 |
| Local REST API Key (插件设置) | Obsidian 插件设置 | 验证谁有权限访问 |

**这两个应该是同一个值**，只是配置在不同的地方。
