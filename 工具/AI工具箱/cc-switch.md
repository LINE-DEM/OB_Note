
## CC-Switch (farion1231) 切换 API 的运行步骤与原理

### 核心原理：一句话概括

CC-Switch **不是代理/中间人**，它本质上是一个**配置文件改写器**。点击"切换"时，它把你选的 Provider 的 API Key、Base URL、Model 等信息直接写入 Claude Code 的配置文件，然后你**重启 Claude Code** 新配置才会生效。

可以同时在不同的软件 使用订阅或者api
步骤 先使用pro 订阅 打开两个软件终端claude
然后cc切换为api
重启其中一个软件即可

所以 可以写一个脚本

---

### 你的场景：从"国内中转 API" 切换到 "MiniMax API"

假设你在 CC-Switch 里有两个 Provider：

**Provider A（国内中转）：**

```json
{
  "ANTHROPIC_AUTH_TOKEN": "sk-zhongzhuan-xxx",
  "ANTHROPIC_BASE_URL": "https://your-relay.com/v1"
}
```

**Provider B（MiniMax）：**

```json
{
  "ANTHROPIC_AUTH_TOKEN": "你的minimax-key",
  "ANTHROPIC_BASE_URL": "https://api.minimaxi.com/anthropic",
  "ANTHROPIC_MODEL": "MiniMax-M2.1"
}
```

---

### 点击切换后，发生了什么（按时间顺序）

**第 1 步：CC-Switch 读取你选择的 Provider 配置**

从它自己的 SSOT（Single Source of Truth）文件 `~/.cc-switch/config.json` 中取出 MiniMax 这个 Provider 的全部参数。

**第 2 步：原子写入 Claude Code 的配置文件**

CC-Switch 把 MiniMax 的配置写入 Claude Code 的实际配置文件：

```
~/.claude/settings.json
```

写入内容类似：

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "你的minimax-key",
    "ANTHROPIC_BASE_URL": "https://api.minimaxi.com/anthropic",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "MiniMax-M2.1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "MiniMax-M2.1",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "MiniMax-M2.1",
    "ANTHROPIC_MODEL": "MiniMax-M2.1"
  }
}
```

写入方式是**原子写入**（先写临时文件，再 rename 覆盖），防止写到一半崩溃导致配置损坏。

**第 3 步：你需要重启 Claude Code**

这是关键——**切换不是热生效的**。Claude Code 在启动时读取 `settings.json`，运行中不会重新加载。所以你必须：

1. 退出当前正在运行的 Claude Code 会话（`/exit` 或 Ctrl+C）
2. 重新启动 `claude` 命令

新的 Claude Code 进程启动时会读取已被改写的 `settings.json`，从而使用 MiniMax 的 API 端点和密钥。

---

### 架构图示

```
┌─────────────────────┐
│  CC-Switch GUI      │
│  (Tauri 桌面应用)    │
│                     │
│  ~/.cc-switch/      │  ← 所有 Provider 存在这里（SSOT）
│  config.json        │
└────────┬────────────┘
         │ 点击"切换到 MiniMax"
         ▼
┌─────────────────────┐
│  原子写入            │
│  ~/.claude/          │
│  settings.json       │  ← Claude Code 的实际配置被覆盖
│                     │
│  env.ANTHROPIC_*    │  ← Key/URL/Model 全部替换
└────────┬────────────┘
         │ 你手动重启 claude
         ▼
┌─────────────────────┐
│  Claude Code 进程    │
│  读取 settings.json  │
│  → 连接 minimax API  │
└─────────────────────┘
```

---

### 几个重要细节

1. **正在运行的 Claude Code 会话不受影响**。你点切换的那一刻，已经在跑的 Claude Code 还在用旧的（国内中转）API，直到你退出重启。
    
2. **双向同步（Backfill）**：如果你在 Claude Code 的 settings.json 里手动改了什么，CC-Switch 编辑当前激活的 Provider 时会把实际文件的内容回填回来，不会丢失手动修改。
    
3. **系统托盘快捷切换**也是同样的机制——写文件，然后你需要重启 CLI。
    
4. 如果你系统环境变量里（如 `.zshrc`）也设了 `ANTHROPIC_API_KEY` 等，**环境变量会覆盖** settings.json 的配置，导致切换看起来没生效。这是常见坑。