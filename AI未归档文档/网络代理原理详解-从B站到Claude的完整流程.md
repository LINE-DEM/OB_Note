# 网络代理原理详解：从B站到Claude的完整流程

> [!info] 文档说明
> 本文详细讲解网络代理的工作原理，包括系统代理、TUN模式、规则分流等核心概念，并通过B站和Claude Code的实际案例进行说明。

---

## 📚 目录

- [[#一、网络请求的七层结构]]
- [[#二、无代理情况下的网络访问]]
- [[#三、代理的两种工作模式]]
- [[#四、Clash规则的作用机制]]
- [[#五、实战案例分析]]
- [[#六、常见问题解答]]

---

## 一、网络请求的七层结构

### OSI七层模型（简化版）

```mermaid
graph TB
    subgraph "应用层面"
        A[应用程序<br/>浏览器/Claude Code] --> B[应用层<br/>HTTP/HTTPS协议]
        B --> C[传输层<br/>TCP/UDP]
        C --> D[网络层<br/>IP地址路由]
    end

    subgraph "物理层面"
        D --> E[数据链路层<br/>MAC地址]
        E --> F[物理层<br/>网卡/光纤]
    end

    F --> G[互联网]

    style A fill:#e1f5ff
    style D fill:#fff4e1
    style F fill:#ffe1e1
```

**关键理解**：
- **应用层**：软件发起请求（如Chrome打开bilibili.com）
- **网络层**：决定数据包走哪条路（**代理在这里起作用**）
- **物理层**：真正的网线、WiFi信号

---

## 二、无代理情况下的网络访问

### 场景1：访问B站（bilibili.com）

```mermaid
sequenceDiagram
    participant 浏览器
    participant 系统网络栈
    participant 本地DNS
    participant 运营商网络
    participant B站服务器

    浏览器->>系统网络栈: 1. 我要访问 bilibili.com
    系统网络栈->>本地DNS: 2. 解析域名
    本地DNS-->>系统网络栈: 3. IP: 120.92.108.x
    系统网络栈->>运营商网络: 4. 发送HTTP请求
    运营商网络->>B站服务器: 5. 转发到国内服务器
    B站服务器-->>运营商网络: 6. 返回视频数据
    运营商网络-->>浏览器: 7. 快速响应（国内路由）

    Note over 浏览器,B站服务器: ✅ 延迟 10-50ms（国内直连）
```

**路径特点**：
- 🚀 **直连**：数据不绕路，走最短路径
- ⚡ **速度快**：国内服务器，延迟低
- 📍 **路由**：运营商智能选择最优线路

---

### 场景2：访问Claude（claude.ai）

```mermaid
sequenceDiagram
    participant Claude软件
    participant 系统网络栈
    participant 本地DNS
    participant 运营商网络
    participant GFW
    participant Claude服务器

    Claude软件->>系统网络栈: 1. 我要访问 claude.ai
    系统网络栈->>本地DNS: 2. 解析域名
    本地DNS-->>系统网络栈: 3. IP: 104.18.x.x (国外)
    系统网络栈->>运营商网络: 4. 发送请求
    运营商网络->>GFW: 5. 检测到国外流量
    GFW->>Claude软件: ❌ 连接被阻断

    Note over Claude软件,Claude服务器: ❌ 无法访问（被墙）
```

**问题**：
- 🚫 **GFW阻断**：防火墙识别并拦截
- ❌ **直连失败**：无法到达目标服务器
- 🔧 **解决方案**：需要代理绕过封锁

---

## 三、代理的两种工作模式

### 模式对比表

| 特性              | 系统代理模式       | TUN虚拟网卡模式    |
| --------------- | ------------ | ------------ |
| **工作层级**        | 应用层（Layer 7） | 网络层（Layer 3） |
| **拦截点**         | 应用主动查询       | 系统网络栈自动拦截    |
| **应用支持**        | 需要应用支持代理协议   | **所有应用透明支持** |
| **Claude Code** | ❌ 可能不生效      | ✅ 一定生效       |
| **终端工具**        | 需要设置环境变量     | ✅ 自动生效       |
| **配置难度**        | ⭐⭐⭐          | ⭐            |

---

### 3.1 系统代理模式详解

#### 工作流程图

```mermaid
graph TB
    A[应用程序启动] --> B{应用是否支持代理?}
    B -->|是| C[读取系统代理设置<br/>127.0.0.1:7897]
    B -->|否| D[直接发送请求<br/>❌ 不走代理]

    C --> E[将请求发送到<br/>本地代理客户端]
    E --> F[Clash/V2Ray<br/>代理工具]
    F --> G[加密并转发到<br/>代理服务器]
    G --> H[代理服务器]
    H --> I[目标网站]

    D --> J[直接访问<br/>可能被墙]

    style C fill:#d4edda
    style D fill:#f8d7da
    style F fill:#fff3cd
```

#### 系统代理设置位置

**Windows**：
```
控制面板 → 网络和Internet → 代理设置
- HTTP代理：127.0.0.1:7897
- HTTPS代理：127.0.0.1:7897
```

**问题案例**：
```
✅ Chrome浏览器：会读取系统代理 → ✅ 生效
❌ Claude Code：不读取系统代理 → ❌ 不生效
❌ 某些终端工具：不读取系统代理 → ❌ 不生效
```

---

### 3.2 TUN虚拟网卡模式详解

#### 工作流程图

```mermaid
graph TB
    subgraph "应用层"
        A1[Chrome浏览器]
        A2[Claude Code]
        A3[终端 curl]
    end

    A1 --> B[发送网络请求]
    A2 --> B
    A3 --> B

    B --> C[系统网络栈<br/>准备发送数据包]

    C --> D{路由表检查}
    D -->|匹配到TUN设备| E[📍 TUN虚拟网卡<br/>clash-tun0]

    E --> F[Clash接管流量<br/>所有应用透明]

    F --> G{检查代理规则}
    G -->|国内IP/域名| H1[DIRECT<br/>直连发送]
    G -->|国外IP/域名| H2[PROXY<br/>加密转发]

    H1 --> I1[运营商网络<br/>国内服务器]
    H2 --> I2[代理服务器<br/>国外节点]

    I2 --> I3[目标国外网站]

    style E fill:#d4edda
    style F fill:#fff3cd
    style G fill:#cfe2ff
```

#### TUN模式的魔法

**虚拟网卡是什么？**
```
物理网卡（真实存在）：
├─ WiFi网卡：wlan0
├─ 有线网卡：eth0
└─ 蓝牙网卡：bluetooth0

虚拟网卡（软件模拟）：
└─ TUN设备：clash-tun0  ← Clash创建的"假网卡"
```

**路由表劫持**：
```bash
# 正常路由表
默认网关 → 192.168.1.1 (路由器)

# TUN模式启动后
默认网关 → clash-tun0 (虚拟网卡) ← 优先级更高！
备用路由 → 192.168.1.1 (路由器)
```

**流程解释**：
1. **所有应用**发送网络请求
2. 系统网络栈查路由表 → 发现 TUN 设备优先级最高
3. 数据包被"劫持"到 Clash 软件
4. Clash 根据**规则**决定：直连 or 代理
5. 应用完全不知道中间发生了什么（**透明代理**）

---

## 四、Clash规则的作用机制

### 4.1 规则工作位置图

```mermaid
graph LR
    A[应用发起请求<br/>bilibili.com] --> B[TUN虚拟网卡<br/>接收流量]

    B --> C{Clash规则引擎}

    C --> D1[规则1: DOMAIN-SUFFIX,cn,DIRECT]
    C --> D2[规则2: GEOIP,CN,DIRECT]
    C --> D3[规则3: DOMAIN-KEYWORD,google,PROXY]
    C --> D4[规则4: MATCH,PROXY]

    D1 --> E{匹配成功?}
    D2 --> E
    D3 --> E
    D4 --> E

    E -->|是 - 规则1| F1[DIRECT<br/>直连运营商]
    E -->|是 - 规则3| F2[PROXY<br/>代理节点]

    F1 --> G1[国内服务器]
    F2 --> G2[国外服务器]

    style C fill:#fff3cd
    style D1 fill:#d4edda
    style D3 fill:#cfe2ff
```

### 4.2 规则类型详解

#### 规则格式

```yaml
规则类型, 匹配内容, 动作
  ↓        ↓       ↓
DOMAIN-SUFFIX, cn, DIRECT
```

#### 常用规则类型

| 规则类型 | 说明 | 示例 | 匹配结果 |
|---------|------|------|---------|
| **DOMAIN-SUFFIX** | 域名后缀匹配 | `DOMAIN-SUFFIX,cn,DIRECT` | `baidu.cn` ✅<br/>`taobao.cn` ✅ |
| **DOMAIN-KEYWORD** | 域名包含关键词 | `DOMAIN-KEYWORD,google,PROXY` | `google.com` ✅<br/>`google.co.jp` ✅ |
| **GEOIP** | IP地理位置 | `GEOIP,CN,DIRECT` | 中国IP段 ✅ |
| **IP-CIDR** | IP地址段 | `IP-CIDR,192.168.0.0/16,DIRECT` | 局域网 ✅ |
| **MATCH** | 兜底规则（匹配所有） | `MATCH,PROXY` | 其他所有流量 |

#### 动作类型

| 动作 | 说明 | 效果 |
|------|------|------|
| **DIRECT** | 直连 | 不走代理，直接连接目标 |
| **PROXY** | 代理 | 通过代理节点转发 |
| **REJECT** | 拒绝 | 拦截连接（常用于广告） |

---

### 4.3 规则执行顺序

> [!important] 关键概念
> Clash规则是**从上到下**依次匹配，**匹配到第一条后立即执行**，不再继续往下。

#### 示例配置

```yaml
rules:
  # 1️⃣ 局域网直连（最高优先级）
  - DOMAIN-SUFFIX,local,DIRECT
  - IP-CIDR,192.168.0.0/16,DIRECT

  # 2️⃣ 国内域名直连
  - DOMAIN-SUFFIX,cn,DIRECT
  - DOMAIN-KEYWORD,baidu,DIRECT
  - DOMAIN-KEYWORD,bilibili,DIRECT
  - DOMAIN-KEYWORD,taobao,DIRECT

  # 3️⃣ 国外常用服务走代理
  - DOMAIN-KEYWORD,google,PROXY
  - DOMAIN-KEYWORD,youtube,PROXY
  - DOMAIN-KEYWORD,claude,PROXY
  - DOMAIN-KEYWORD,github,PROXY

  # 4️⃣ 地理位置规则
  - GEOIP,CN,DIRECT        # 中国IP直连

  # 5️⃣ 兜底规则（最后）
  - MATCH,PROXY            # 其他所有流量走代理
```

#### 匹配流程示例

```mermaid
graph TD
    A[请求: bilibili.com] --> B{规则1: local?}
    B -->|否| C{规则2: 192.168.x.x?}
    C -->|否| D{规则3: *.cn?}
    D -->|否| E{规则4: baidu?}
    E -->|否| F{规则5: bilibili?}
    F -->|✅ 是| G[执行: DIRECT<br/>直连访问]

    G --> H[✅ 结果: 快速访问B站]

    style F fill:#d4edda
    style G fill:#d4edda
    style H fill:#d4edda
```

```mermaid
graph TD
    A[请求: claude.ai] --> B{规则1-5: 国内相关?}
    B -->|否| C{规则6: google?}
    C -->|否| D{规则7: youtube?}
    D -->|否| E{规则8: claude?}
    E -->|✅ 是| F[执行: PROXY<br/>走代理节点]

    F --> G[✅ 结果: 成功访问Claude]

    style E fill:#cfe2ff
    style F fill:#cfe2ff
    style G fill:#cfe2ff
```

---

## 五、实战案例分析

### 案例1：访问B站（bilibili.com）

#### 全局模式（您当前遇到的问题）

```mermaid
sequenceDiagram
    participant 浏览器
    participant TUN虚拟网卡
    participant Clash
    participant 日本节点
    participant B站服务器

    浏览器->>TUN虚拟网卡: 1. 访问 bilibili.com
    TUN虚拟网卡->>Clash: 2. 拦截流量

    Note over Clash: 全局模式：<br/>所有流量强制走PROXY

    Clash->>日本节点: 3. 加密转发
    日本节点->>B站服务器: 4. 从日本访问中国
    B站服务器-->>日本节点: 5. 返回数据
    日本节点-->>浏览器: 6. 转发回来

    Note over 浏览器,B站服务器: ❌ 延迟 200ms+（绕路日本）
    Note over 浏览器,B站服务器: 🐌 速度慢、可能限速
```

**问题根源**：
- 国内网站也绕到国外 → 延迟增加
- B站可能检测到国外IP → 限速或阻断
- 浪费代理流量

---

#### 规则模式（推荐配置）

```mermaid
sequenceDiagram
    participant 浏览器
    participant TUN虚拟网卡
    participant Clash
    participant 运营商
    participant B站服务器

    浏览器->>TUN虚拟网卡: 1. 访问 bilibili.com
    TUN虚拟网卡->>Clash: 2. 拦截流量

    Note over Clash: 规则匹配：<br/>DOMAIN-KEYWORD,bilibili,DIRECT

    Clash->>运营商: 3. 直连发送
    运营商->>B站服务器: 4. 国内直连
    B站服务器-->>运营商: 5. 快速返回
    运营商-->>浏览器: 6. 直接送达

    Note over 浏览器,B站服务器: ✅ 延迟 10-30ms（直连）
    Note over 浏览器,B站服务器: 🚀 速度快、不限速
```

**规则配置**：
```yaml
rules:
  - DOMAIN-KEYWORD,bilibili,DIRECT
  - DOMAIN-SUFFIX,bilibili.com,DIRECT
  - GEOIP,CN,DIRECT
```

---

### 案例2：访问Claude Code（claude.ai）

#### 系统代理模式（不生效）

```mermaid
sequenceDiagram
    participant Claude软件
    participant 系统网络栈
    participant GFW
    participant Claude服务器

    Claude软件->>系统网络栈: 1. 我要访问 claude.ai

    Note over Claude软件: Claude Code不支持代理协议<br/>没有读取127.0.0.1:7897

    系统网络栈->>GFW: 2. 直接发送请求
    GFW->>Claude软件: ❌ 连接被阻断

    Note over Claude软件,Claude服务器: ❌ 无法访问
```

**原因**：
- Claude Code 基于 Electron
- 没有实现系统代理协议读取
- 直接发送请求 → 被墙

---

#### TUN模式（成功）

```mermaid
sequenceDiagram
    participant Claude软件
    participant TUN虚拟网卡
    participant Clash
    participant 美国节点
    participant Claude服务器

    Claude软件->>TUN虚拟网卡: 1. 访问 claude.ai
    TUN虚拟网卡->>Clash: 2. 自动拦截

    Note over Clash: 规则匹配：<br/>DOMAIN-KEYWORD,claude,PROXY

    Clash->>美国节点: 3. 加密转发
    美国节点->>Claude服务器: 4. 从美国访问
    Claude服务器-->>美国节点: 5. 返回数据
    美国节点-->>Claude软件: 6. 解密送达

    Note over Claude软件,Claude服务器: ✅ 成功访问（透明代理）
```

**规则配置**：
```yaml
rules:
  - DOMAIN-KEYWORD,claude,PROXY
  - DOMAIN-KEYWORD,anthropic,PROXY
  - DOMAIN-SUFFIX,claude.ai,PROXY
```

---

### 案例3：完整的混合场景

#### 同时访问国内外网站

```mermaid
graph TB
    subgraph "应用层"
        A1[Chrome: bilibili.com]
        A2[Chrome: youtube.com]
        A3[Claude Code: claude.ai]
    end

    A1 --> B[TUN虚拟网卡]
    A2 --> B
    A3 --> B

    B --> C{Clash规则引擎}

    C -->|匹配: bilibili → DIRECT| D1[直连]
    C -->|匹配: youtube → PROXY| D2[代理]
    C -->|匹配: claude → PROXY| D3[代理]

    D1 --> E1[运营商网络<br/>B站服务器<br/>⚡ 10ms]

    D2 --> E2[香港节点<br/>YouTube服务器<br/>⚡ 50ms]

    D3 --> E3[美国节点<br/>Claude服务器<br/>⚡ 150ms]

    style D1 fill:#d4edda
    style D2 fill:#cfe2ff
    style D3 fill:#cfe2ff
```

**效果**：
- ✅ B站：直连快速
- ✅ YouTube：代理访问
- ✅ Claude：代理访问
- ✅ 所有应用都生效（TUN模式）

---

## 六、常见问题解答

### Q1：为什么全局模式下访问国内网站慢？

> [!warning] 原因
> **全局模式 = 强制所有流量走代理**

```
您的网络 → 日本节点 → B站（中国）
   ↑                              ↓
   └──────── 绕了一大圈 ──────────┘
```

**解决方案**：切换到**规则模式**，让国内流量直连。

---

### Q2：规则模式和全局模式有什么区别？

| 模式 | 流量路由 | 适用场景 |
|------|---------|---------|
| **规则模式** | 根据规则智能分流 | 日常使用（推荐） |
| **全局模式** | 所有流量强制走代理 | 临时测试、特殊需求 |
| **直连模式** | 所有流量都不走代理 | 不需要代理时 |

---

### Q3：为什么设置了系统代理，Claude Code还是不生效？

> [!info] 技术原因
> **系统代理 = 应用层协议**，需要应用主动实现。

**不支持的应用**：
- Claude Code（Electron，未实现）
- 某些终端工具（不读取系统代理）
- 游戏客户端（直连网络栈）

**解决方案**：
1. 使用 **TUN模式**（一劳永逸）
2. 设置**环境变量**（每次启动前）
3. 使用**启动脚本**（自动化）

---

### Q4：TUN模式会影响游戏延迟吗？

> [!tip] 配置建议
> 在规则中添加游戏服务器直连规则。

```yaml
rules:
  # 游戏服务器直连
  - DOMAIN-SUFFIX,riotgames.com,DIRECT     # 英雄联盟
  - DOMAIN-SUFFIX,pubg.com,DIRECT          # 吃鸡
  - IP-CIDR,103.10.124.0/24,DIRECT         # 游戏服务器IP段
```

**效果**：
- ✅ 游戏流量直连 → 低延迟
- ✅ 网页浏览智能分流 → 快速访问

---

### Q5：我的配置文件应该怎么写？

#### 推荐配置模板

```yaml
# Clash Verge 配置文件

mode: rule  # 使用规则模式
allow-lan: true
log-level: info

# DNS配置（重要！）
dns:
  enable: true
  enhanced-mode: fake-ip
  nameserver:
    - 223.5.5.5       # 阿里DNS
    - 119.29.29.29    # 腾讯DNS
  fallback:
    - 1.1.1.1         # Cloudflare
    - 8.8.8.8         # Google

# TUN模式配置
tun:
  enable: true
  stack: system
  dns-hijack:
    - any:53

# 规则配置
rules:
  # 1. 局域网直连
  - DOMAIN-SUFFIX,local,DIRECT
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT

  # 2. 国内常用网站直连
  - DOMAIN-KEYWORD,baidu,DIRECT
  - DOMAIN-KEYWORD,taobao,DIRECT
  - DOMAIN-KEYWORD,jd,DIRECT
  - DOMAIN-KEYWORD,bilibili,DIRECT
  - DOMAIN-KEYWORD,qq,DIRECT
  - DOMAIN-KEYWORD,aliyun,DIRECT
  - DOMAIN-SUFFIX,cn,DIRECT

  # 3. 国外服务走代理
  - DOMAIN-KEYWORD,google,PROXY
  - DOMAIN-KEYWORD,youtube,PROXY
  - DOMAIN-KEYWORD,github,PROXY
  - DOMAIN-KEYWORD,claude,PROXY
  - DOMAIN-KEYWORD,openai,PROXY

  # 4. 地理位置规则
  - GEOIP,CN,DIRECT

  # 5. 兜底规则
  - MATCH,PROXY
```

---

## 七、完整工作流程总结

### 终极流程图

```mermaid
graph TB
    A[您打开应用] --> B{发起网络请求}

    B --> C[系统网络栈]

    C --> D{是否开启TUN模式?}

    D -->|否| E{应用是否支持代理?}
    E -->|是| F[读取系统代理<br/>127.0.0.1:7897]
    E -->|否| G[直接发送请求<br/>❌ 可能被墙]

    D -->|是| H[TUN虚拟网卡<br/>接管所有流量]

    F --> I[Clash代理客户端]
    H --> I

    I --> J{Clash工作模式?}

    J -->|全局模式| K[强制所有流量<br/>走代理节点]
    J -->|规则模式| L{匹配规则}
    J -->|直连模式| M[所有流量直连]

    L -->|国内域名/IP| N[DIRECT<br/>直连运营商]
    L -->|国外域名/IP| O[PROXY<br/>代理节点]

    N --> P[国内服务器<br/>✅ 快速访问]
    O --> Q[国外节点<br/>✅ 突破封锁]
    K --> Q

    style H fill:#d4edda
    style L fill:#fff3cd
    style P fill:#d4edda
    style Q fill:#cfe2ff
```

---

## 八、关键概念速查表

| 概念          | 简单理解               | 技术层级         |
| ----------- | ------------------ | ------------ |
| **系统代理**    | 应用主动问："我该走哪里？"     | 应用层（Layer 7） |
| **TUN虚拟网卡** | 系统强制劫持："你必须走这里！"   | 网络层（Layer 3） |
| **全局模式**    | 所有流量都走代理（绕路）       | Clash策略      |
| **规则模式**    | 根据规则智能分流（推荐）       | Clash策略      |
| **直连模式**    | 所有流量都不走代理          | Clash策略      |
| **DIRECT**  | 不经过代理，直接访问         | Clash规则动作    |
| **PROXY**   | 通过代理节点转发           | Clash规则动作    |
| **规则**      | 告诉Clash："这个网站该怎么走" | Clash配置      |

---

## 九、推荐配置方案

### 方案：TUN模式 + 规则分流

```yaml
✅ 优势：
- 所有应用自动生效（浏览器、Claude Code、终端）
- 国内网站快速直连
- 国外网站智能代理
- 无需手动配置环境变量

🎯 适用场景：
- 日常使用
- 开发工作
- 混合访问国内外网站
```

---

## 十、参考资源

- [[网络基础知识]]
- [[Clash配置详解]]
- [[代理协议对比]]

---

> [!success] 学习检查
> 阅读完本文后，您应该能够理解：
> - ✅ 为什么全局模式下访问国内网站慢
> - ✅ TUN模式和系统代理的区别
> - ✅ Clash规则是如何工作的
> - ✅ 如何配置规则实现智能分流
> - ✅ 为什么Claude Code需要TUN模式

---

**最后更新**：2026-01-29
**作者**：Craft Agent
**标签**：#网络原理 #代理 #Clash #TUN模式 #技术教程
