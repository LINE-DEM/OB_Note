### PowerShell vs Bash 常用命令

| 任务 | PowerShell | Bash |
|------|-----------|------|
| 列出文件 | `ls` / `Get-ChildItem` | `ls` |
| 查看内容 | `cat` / `Get-Content` | `cat` |
| 查找文本 | `Select-String` | `grep` |
| 下载文件 | `Invoke-WebRequest` | `curl` / `wget` |
| 查看进程 | `Get-Process` | `ps` |
| 杀死进程 | `Stop-Process` | `kill` |
| 环境变量 | `$env:VAR` | `$VAR` |
| 设置变量 | `$var = "value"` | `var="value"` |
# LLM 相关技术名词完全解析

> 为 Claude Code + Codex API 用户准备的通俗易懂指南

---

## 📚 核心概念地图

```
你的使用场景：
┌─────────────────────────────────────────────────────────────┐
│  你（开发者）                                                │
│    ↓ 使用                                                    │
│  Claude Code（工具）                                         │
│    ↓ 需要调用                                                │
│  LLM API（各种 AI 模型服务）                                 │
│    ├─ Anthropic API (Claude)                                │
│    ├─ OpenAI API (Codex/GPT)                                │
│    └─ 其他 API                                               │
│    ↓ 通过                                                    │
│  Proxy/Gateway（中间代理层）← 这里就用到所有这些名词        │
│    ↓ 运行在                                                  │
│  Docker（容器环境）                                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ LLM（Large Language Model）


---

## 2️⃣ Proxy（代理）

### **什么是 Proxy？**

**定义**：代理是一个中间层，帮你转发请求和响应。

### **在你的场景中**

```
没有 Proxy：
Claude Code → 直接调用 Anthropic API
           → 直接调用 OpenAI API
问题：
- 需要分别配置两个 API
- Claude Code 只支持 Anthropic 格式
- 无法使用 OpenAI API ❌

有 Proxy（LiteLLM）：
Claude Code → LiteLLM Proxy → 自动转换格式 → Anthropic API
                            → 自动转换格式 → OpenAI API
                            → 自动转换格式 → 其他 API
好处：
- Claude Code 只需要配置一次
- 自动格式转换
- 可以使用任何 API ✅
```

### **技术细节**

```
Proxy 的三个核心功能：

1. 格式转换
   Claude Code 请求（Anthropic 格式）
     ↓ Proxy 转换
   OpenAI API（OpenAI 格式）

2. 路由选择
   请求：我要最便宜的 LLM
     ↓ Proxy 判断
   转发到：GPT-4o-mini（最便宜）

3. 负载均衡
   100 个请求
     ↓ Proxy 分配
   ├─ 50 个 → API Key 1
   ├─ 30 个 → API Key 2
   └─ 20 个 → API Key 3
```

---

## 3️⃣ LLM 网关（LLM Gateway）

### **什么是 LLM 网关？**

**定义**：LLM 网关是一个增强版的 Proxy，不仅转发请求，还提供管理、监控、优化等功能。

**生活比喻**：

```
Proxy（简单代理）= 小卖部
- 帮你买东西
- 收钱

Gateway（网关）= 大型超市
- 帮你买东西
- 收钱
- 会员管理
- 积分系统
- 库存管理
- 价格优化
- 购物历史记录
```

### **在你的场景中**

```
LiteLLM 就是一个 LLM 网关

功能：
├─ Proxy 功能（基础）
│   ├─ 请求转发
│   └─ 格式转换
│
├─ Gateway 增强功能
│   ├─ 成本追踪（每个请求花了多少钱）
│   ├─ 使用统计（调用了多少次 API）
│   ├─ 缓存优化（相同问题不重复调用）
│   ├─ 速率限制（防止超额）
│   ├─ 故障转移（API A 坏了自动用 API B）
│   ├─ 负载均衡（分散请求到多个 API）
│   └─ 访问控制（谁能用哪个 API）
```

### **Proxy vs Gateway 对比**

```
┌─────────────────┬──────────────┬──────────────┐
│ 功能            │ Proxy        │ Gateway      │
├─────────────────┼──────────────┼──────────────┤
│ 请求转发        │ ✅           │ ✅           │
│ 格式转换        │ ✅           │ ✅           │
│ 成本追踪        │ ❌           │ ✅           │
│ 使用统计        │ ❌           │ ✅           │
│ 缓存            │ ❌           │ ✅           │
│ 故障转移        │ ❌           │ ✅           │
│ 负载均衡        │ ❌           │ ✅           │
│ Web UI 管理     │ ❌           │ ✅           │
└─────────────────┴──────────────┴──────────────┘
```

---

## 4️⃣ Router（核心路由器）

### **什么是 Router？**

**定义**：路由器是网关内部的"大脑"，负责决定每个请求应该发送到哪个 LLM。

**生活比喻**：

```
Router = 餐厅的服务员领班

客人来了：
领班（Router）判断：
├─ 这是 VIP 客人 → 安排最好的厨师（最贵的 LLM）
├─ 这是普通客人 → 安排普通厨师（便宜的 LLM）
├─ 这是简单需求 → 安排学徒（最便宜的 LLM）
└─ A 厨师太忙了 → 安排 B 厨师（负载均衡）
```

### **在你的场景中**

```
你配置了多个 LLM：
├─ Claude Sonnet 4（贵但好）
├─ GPT-4o（中等）
└─ GPT-4o-mini（便宜）

Router 的工作：

请求 1："帮我写一个复杂的算法"
  ↓ Router 判断：需要强大能力
  ↓ 选择：Claude Sonnet 4 ⭐

请求 2："这个变量应该叫什么名字？"
  ↓ Router 判断：简单任务
  ↓ 选择：GPT-4o-mini ⭐（省钱）

请求 3："分析这段代码"
  ↓ Router 判断：中等难度
  ↓ 选择：GPT-4o ⭐
  ↓ 如果 GPT-4o 失败
  ↓ 故障转移：Claude Sonnet 4
```

### **Router 配置示例**

```yaml
# LiteLLM 配置文件

router_settings:
  # 路由策略
  routing_strategy: "cost-based"  # 基于成本
  # 或者
  routing_strategy: "latency-based"  # 基于速度
  # 或者
  routing_strategy: "simple-shuffle"  # 随机分配

  # 故障转移
  fallbacks:
    - claude-sonnet-4: [gpt-4o, gpt-4o-mini]
      # Claude 失败 → 尝试 GPT-4o → 再失败 → GPT-4o-mini

  # 负载均衡
  num_retries: 3  # 失败重试 3 次
  timeout: 60     # 60 秒超时
```

### **路由决策流程**

```
接收请求
  ↓
Router 分析请求：
  - 任务复杂度
  - 成本预算
  - 响应速度要求
  - 当前 API 状态
  ↓
应用路由规则：
  - 简单任务 → 便宜的 LLM
  - 复杂任务 → 强大的 LLM
  - API 限流 → 切换备用 API
  ↓
发送到选定的 LLM
  ↓
返回结果
```

---

## 5️⃣ Docker

### **什么是 Docker？**

**定义**：Docker 是一个容器技术，可以把应用和它需要的所有东西打包在一起。

### **更详细的比喻**

```
没有 Docker：
你要运行 LiteLLM

步骤：
1. 安装 Python
2. 安装一堆依赖库
3. 配置环境变量
4. 解决各种版本冲突
5. 终于可以运行了！
6. 换一台电脑 → 重新来一遍 😭

有了 Docker：
1. docker run litellm
2. 完成！✅
3. 换任何电脑都一样
```

### **在你的场景中**

```
安装 LiteLLM 的两种方式：

方式 A：直接安装（麻烦）
pip install litellm
# 可能遇到的问题：
# - Python 版本不对
# - 依赖库冲突
# - 系统不兼容

方式 B：使用 Docker（推荐）
docker run -p 4000:4000 litellm
# 优势：
# - 一键启动
# - 不污染系统
# - 版本一致
# - 跨平台
```

### **Docker 核心概念**

```
1. Image（镜像）= 食谱
   - 包含所有配置和代码
   - 只读的
   - 例如：litellm:latest

2. Container（容器）= 做出来的菜
   - 从镜像创建的运行实例
   - 可以启动、停止、删除
   - 例如：你运行的 LiteLLM

3. Volume（数据卷）= 保鲜盒
   - 存储数据
   - 容器删了数据还在
   - 例如：配置文件、日志
```

### **实际使用**

```bash
# 下载 LiteLLM 镜像
docker pull ghcr.io/berriai/litellm:main-latest

# 运行容器
docker run -d \
  --name litellm-proxy \
  -p 4000:4000 \
  -v $(pwd)/config.yaml:/app/config.yaml \
  litellm:latest

# 解释：
# -d              后台运行
# --name          容器名字
# -p 4000:4000    端口映射（电脑4000→容器4000）
# -v              挂载配置文件
# litellm:latest  使用的镜像

# 查看运行状态
docker ps

# 查看日志
docker logs litellm-proxy

# 停止
docker stop litellm-proxy

# 删除
docker rm litellm-proxy
```

---

## 6️⃣ Docker Compose

### **什么是 Docker Compose？**

**定义**：Docker Compose 是用来管理多个 Docker 容器的工具。

**生活比喻**：

```
Docker = 开一家店
- 需要手动：租店面、装修、进货、招员工

Docker Compose = 开连锁店
- 用一个标准化流程
- 一次性搞定所有事情
- 所有店铺配置一致
```

### **更具体的例子**

```
你要运行一个完整的系统：
├─ LiteLLM Proxy（代理服务）
├─ PostgreSQL（数据库，存储使用记录）
├─ Redis（缓存，提高速度）
└─ Prometheus（监控，查看统计）

没有 Docker Compose：
docker run litellm ...
docker run postgres ...
docker run redis ...
docker run prometheus ...
# 4 个命令，很麻烦

有了 Docker Compose：
docker-compose up
# 一个命令，全部启动！✅
```

### **在你的场景中**

**创建 `docker-compose.yaml`**：

```yaml
version: '3.8'

services:
  # LiteLLM 代理
  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    container_name: litellm-proxy
    ports:
      - "4000:4000"
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - DATABASE_URL=postgresql://user:pass@postgres:5432/litellm
    volumes:
      - ./litellm_config.yaml:/app/config.yaml
    depends_on:
      - postgres
      - redis
    command: --config /app/config.yaml

  # PostgreSQL 数据库
  postgres:
    image: postgres:15
    container_name: litellm-db
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=litellm
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  # Redis 缓存
  redis:
    image: redis:7-alpine
    container_name: litellm-cache
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  # Prometheus 监控（可选）
  prometheus:
    image: prom/prometheus:latest
    container_name: litellm-metrics
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

volumes:
  postgres-data:
  redis-data:
```

### **使用 Docker Compose**

```bash
# 启动所有服务（一键启动！）
docker-compose up -d

# 输出：
# Creating network "litellm_default"
# Creating litellm-db     ... done
# Creating litellm-cache  ... done
# Creating litellm-proxy  ... done
# Creating litellm-metrics ... done

# 查看状态
docker-compose ps

# 输出：
#     Name              Command          State           Ports
# ------------------------------------------------------------------
# litellm-proxy    litellm --config   Up      0.0.0.0:4000->4000/tcp
# litellm-db       postgres           Up      0.0.0.0:5432->5432/tcp
# litellm-cache    redis-server       Up      0.0.0.0:6379->6379/tcp

# 查看日志
docker-compose logs -f litellm

# 停止所有服务
docker-compose down

# 停止并删除数据
docker-compose down -v
```

---

## 🔗 所有概念的关系图

```
┌─────────────────────────────────────────────────────────────┐
│  你（开发者）                                                │
│    ↓                                                         │
│  Claude Code                                                │
│    │                                                         │
│    │ 发送请求（Anthropic 格式）                              │
│    ↓                                                         │
├─────────────────────────────────────────────────────────────┤
│  🐳 Docker Container（容器）                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  LLM Gateway (LiteLLM)                                │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  Proxy（代理层）                                 │  │  │
│  │  │  ├─ 接收请求                                    │  │  │
│  │  │  ├─ 格式转换                                    │  │  │
│  │  │  └─ 身份验证                                    │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  Router（路由器）⭐ 核心大脑                    │  │  │
│  │  │  ├─ 分析请求特征                                │  │  │
│  │  │  ├─ 应用路由规则                                │  │  │
│  │  │  ├─ 选择最佳 LLM                                │  │  │
│  │  │  └─ 处理故障转移                                │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  增强功能                                        │  │  │
│  │  │  ├─ 成本追踪                                    │  │  │
│  │  │  ├─ 使用统计                                    │  │  │
│  │  │  ├─ 缓存                                        │  │  │
│  │  │  └─ 监控                                        │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│    │                                                         │
│    │ Router 根据配置文件规则选择                             │
│    ↓                                                         │
├─────────────────────────────────────────────────────────────┤
│  LLM 后端（多个选择）                                        │
│  ┌─────────────────┬─────────────────┬─────────────────┐    │
│  │ Anthropic API   │ OpenAI API      │ 其他 API        │    │
│  │ (Claude)        │ (GPT-4, Codex)  │ (Gemini等)      │    │
│  └─────────────────┴─────────────────┴─────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 实际应用场景

### **场景：你要同时使用 Claude 和 Codex**

```
第 1 步：安装 Docker
# macOS
brew install docker

# Windows
# 下载 Docker Desktop

第 2 步：创建配置文件
# litellm_config.yaml
model_list:
  - model_name: claude-sonnet-4-20250514
    litellm_params:
      model: anthropic/claude-sonnet-4-20250514
      api_key: os.environ/ANTHROPIC_API_KEY
  
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY

router_settings:
  routing_strategy: simple-shuffle
  fallbacks:
    - claude-sonnet-4-20250514: [gpt-4o]

第 3 步：使用 Docker 启动 LiteLLM
docker run -d \
  --name litellm-proxy \
  -p 4000:4000 \
  -e ANTHROPIC_API_KEY=sk-ant-... \
  -e OPENAI_API_KEY=sk-... \
  -v $(pwd)/litellm_config.yaml:/app/config.yaml \
  ghcr.io/berriai/litellm:main-latest \
  --config /app/config.yaml

第 4 步：配置 Claude Code
export ANTHROPIC_BASE_URL="http://localhost:4000"
export ANTHROPIC_API_KEY="any-key"

第 5 步：使用
claude
# 现在 Claude Code 可以同时使用 Claude 和 Codex API！
```

---

## 🎓 总结

### **名词关系总结**

```
LLM
  └─ 是什么：AI 模型（Claude、GPT-4 等）

Proxy
  └─ 是什么：请求转发的中间层
  └─ 作用：连接 Claude Code 和各种 LLM API

LLM Gateway
  └─ 是什么：增强版 Proxy
  └─ 作用：Proxy + 管理 + 监控 + 优化

Router
  └─ 是什么：Gateway 的核心部件
  └─ 作用：决定请求发送到哪个 LLM

Docker
  └─ 是什么：容器技术
  └─ 作用：打包和运行 LiteLLM（Gateway）

Docker Compose
  └─ 是什么：多容器管理工具
  └─ 作用：一键启动 LiteLLM + 数据库 + 缓存等
```

### **核心工作流程**

```
你的请求
  ↓
Claude Code
  ↓
LiteLLM Proxy（运行在 Docker 容器中）
  ├─ 接收请求
  ├─ 格式转换
  ├─ Router 选择最佳 LLM
  ├─ 转发请求
  ├─ 接收响应
  ├─ 格式转换回来
  └─ 返回给 Claude Code
  ↓
你收到结果
```

### **关键理解**

1. **LLM** = AI 模型本身
2. **Proxy** = 转发请求的中间人
3. **Gateway** = 功能强大的 Proxy
4. **Router** = 决策中心
5. **Docker** = 运行环境
6. **Docker Compose** = 一键管理

---

**记住**：这些都是为了让你能在 Claude Code 中同时使用多个 LLM API（包括 Codex）而需要的技术组件！