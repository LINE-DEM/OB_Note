
### 快速时间线梳理：

 失去焦点 导致unity程序暂停 此时Unity程序的WebSocket无法回复任何消息  服务器的消息发送来后 Unity的WebSocket无法发送任何消息 导致服务器认为连接断开了 服务器持续发送Ping多次无果后断开连接 unity焦点回归后 自动重连





### 可能的Bug：

在当前实现下，Unity 失焦一段时间再回焦，会出现以下高风险行为：
1. **如果服务端每次连接都会初始化会话状态，客户端却以为是续跑，会发生状态错位**：表现包括语言/上下文丢失、对话 session 被重置、conversation_id 关联失效、服务端资源重复初始化等。
2. **回焦时可能触发重连风暴**：`NetWorkMono` 的重连与发送都可能触发 `InitializeWebSocket()`，且缺少“正在连接”的互斥控制，回焦后容易出现**多次并发 ConnectAsync**，服务端会看到多条新连接。循环触发连接 的同时 发生消息触发连接






---
### 排查方向：

1.先确认发生不回复BUG的时候 后端的日志是否收到消息 如果收到 就能证明我这边重新连接成功 发消息成功 需要后端思考 我重新建立连接后 后端的状态是否会混乱（比如conversation_id）
可以再次尝试复现 

2.发送消息等对话id=3的时候  服务器开始回复的时候 失去unity焦点 等待服务器断开 此时打印焦点返回时间 监听网络状态 需要在网络还没连接成功的时候（or 网络data.json还没发给我的时 服务器还没初始化好的时候）发送对话消息 检查 服务器的对话id是否会重置状态






### 代码层次：

```
第 9 层: ClientWebSocket       ← 你的代码在这里
第 8 层: ManagedWebSocket      ← ⭐ Ping/Pong 在这里！
第 7 层: HTTP                  ← 只用于握手
第 6 层: NetworkStream         ← 流抽象
第 5 层: Socket                ← .NET 封装
第 4 层: SocketPal             ← 系统调用封装
第 3 层: 系统调用 API          ← connect()/send()/recv()
第 2 层: TCP                   ← ⭐ 三次握手在这里！
第 1 层: IP                    ← 路由寻址
第 0 层: 链路层/物理层         ← 网卡硬件
```

---



## 📊 问题场景重现

```
时间轴：完整的断连过程

T0 (00:00:00)  Unity 正常运行，WebSocket 连接正常
               ├─ Unity 主线程运行
               ├─ WebSocket 接收线程运行
               └─ TCP 连接 ESTABLISHED

T1 (00:00:05)  ⚠️ Unity 失去焦点（用户切换窗口）
               ├─ Unity 触发 OnApplicationPause(true)
               ├─ ⭐ Unity 主线程暂停
               ├─ ⚠️ Update/FixedUpdate 停止执行
               ├─ ❓ WebSocket 接收线程状态？
               └─ TCP 连接仍然是 ESTABLISHED（操作系统层）

T2 (00:00:10)  服务器发送数据（聊天消息）
               服务器 → 客户端：
                 Data Frame: "你好"
                 ─────────────────────────→
               
               ⭐ 数据到达 Unity 电脑的网卡
               ⭐ 操作系统 TCP 栈接收数据
               ⭐ 数据进入 Socket 接收缓冲区
               ❌ 但 Unity 线程暂停，无法读取！

T3 (00:00:40)  ⏰ 服务器 KeepAlive 定时器触发
               服务器发送 Ping：
                 Ping Frame (Opcode=0x9)
                 ─────────────────────────→
               
               ⭐ Ping 到达 Unity 的 Socket 缓冲区
               ❌ Unity 线程暂停，无法处理
               ❌ 无法自动回复 Pong

T4 (00:00:45)  ⏰ 服务器 Pong 超时检测
               服务器心想：
                 "5 秒了，还没收到 Pong？"
                 ↓
               再发一次 Ping
                 Ping Frame
                 ─────────────────────────→
               
               ❌ 还是没有 Pong

T5 (00:00:50)  ⏰ 服务器第二次 Pong 超时
               服务器心想：
                 "10 秒了，连续 2 次 Ping 没响应"
                 "客户端可能挂了！"
                 ↓
               服务器主动关闭连接：
                 Close Frame (1001 - Going Away)
                 ─────────────────────────→
               
               服务器执行：
                 socket.Close()
                 ↓
               发送 TCP FIN 包
                 FIN
                 ─────────────────────────→

T6 (00:00:51)  ⭐ 操作系统层的 TCP 四次挥手
               服务器 → Unity 操作系统：
                 FIN (我要关闭了)
                 ─────────────────────────→
               
               Unity 操作系统自动回复：
                 ACK (收到你的 FIN)
                 ←─────────────────────────
               
               Unity 操作系统自己也发 FIN：
                 FIN (我也关闭)
                 ←─────────────────────────
               
               服务器操作系统回复：
                 ACK (收到你的 FIN)
                 ─────────────────────────→
               
               ✅ TCP 连接完全关闭（操作系统层）
               ❌ 但 Unity 应用层还不知道！

T7 (00:01:00)  👤 用户切回 Unity 窗口
               Unity 恢复焦点：
                 OnApplicationPause(false)
                 ↓
               Unity 主线程恢复运行
                 Update() 开始执行
                 ↓
               WebSocket 接收线程尝试读取：
                 await socket.ReceiveAsync(...)
                 ↓
               ❌ Socket 已关闭！
                 抛出异常：SocketException
                 "An existing connection was forcibly 
                  closed by the remote host"

T8 (00:01:01)  🔄 Unity 自动重连逻辑触发
               捕获异常 → 调用重连函数
                 ↓
               创建新的 WebSocket 连接
                 new ClientWebSocket()
                 await ws.ConnectAsync(uri)
                 ↓
               TCP 三次握手
                 SYN →
                 ← SYN-ACK
                 ACK →
                 ↓
               HTTP Upgrade 握手
                 GET /ws HTTP/1.1
                 Upgrade: websocket
                 ─────────────────────────→
                 ←─────────────────────────
                 HTTP/1.1 101 Switching Protocols
                 ↓
               ✅ 新连接建立成功
```

---















## 🔍 深入分析：底层发生了什么

### **问题 1：为什么只在服务器发消息时看到 Ping/Pong/Close？**

#### **根本原因：Unity 线程暂停 vs 操作系统继续工作**

```
┌─────────────────────────────────────────────────────────────┐
│  应用层 (Unity C# 代码)                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  WebSocket 接收循环:                                   │ │
│  │  while (true) {                                        │ │
│  │      await socket.ReceiveAsync(buffer);  ⚠️ 暂停在这里 │ │
│  │      ProcessFrame(buffer);                             │ │
│  │  }                                                      │ │
│  └────────────────────────────────────────────────────────┘ │
│  状态：❌ 线程暂停，无法执行任何代码                          │
└─────────────────────────────────────────────────────────────┘
                            ↓ 无法读取
┌─────────────────────────────────────────────────────────────┐
│  .NET Socket 层                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Socket 接收缓冲区 (Receive Buffer)                    │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │ [Ping Frame] [Ping Frame] [Close Frame] [FIN]   │ │ │
│  │  │  ↑ 数据堆积在这里                                │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│  状态：✅ 缓冲区有数据，等待应用层读取                        │
└─────────────────────────────────────────────────────────────┘
                            ↑ 数据到达
┌─────────────────────────────────────────────────────────────┐
│  操作系统 TCP/IP 栈                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  TCP 连接状态：ESTABLISHED → FIN_WAIT → CLOSED        │ │
│  │                                                         │ │
│  │  收到服务器的数据：                                     │ │
│  │  1. WebSocket Ping Frame  ← 正常接收                  │ │
│  │  2. WebSocket Ping Frame  ← 正常接收                  │ │
│  │  3. WebSocket Close Frame ← 正常接收                  │ │
│  │  4. TCP FIN               ← 触发四次挥手               │ │
│  │                                                         │ │
│  │  自动执行 TCP 四次挥手：                                │ │
│  │  收到 FIN → 回 ACK → 发 FIN → 收到 ACK → CLOSED       │ │
│  └────────────────────────────────────────────────────────┘ │
│  状态：✅ 完全自动，不需要应用层参与                          │
└─────────────────────────────────────────────────────────────┘
```

#### **关键点解析**

**① 数据确实到达了 Unity 电脑**

```
服务器发送的每一个包（Ping/Close/FIN）都：
✅ 成功到达 Unity 电脑的网卡
✅ 被操作系统 TCP/IP 栈接收
✅ 放入 Socket 接收缓冲区

但是：
❌ Unity 应用层线程暂停
❌ 无法调用 socket.ReceiveAsync() 读取数据
❌ 数据堆积在缓冲区，无人处理
```

**② TCP 层自动处理关闭**

```csharp
// 操作系统内核的行为（自动执行，不需要应用参与）

// 收到服务器的 FIN 包
OnReceiveTcpFin() 
{
    // 1. 自动回复 ACK
    SendTcpAck();
    
    // 2. 改变连接状态
    state = FIN_WAIT_2;
    
    // 3. 自己也发送 FIN
    SendTcpFin();
    
    // 4. 等待对方的 ACK
    WaitForAck();
    
    // 5. 连接关闭
    state = CLOSED;
    
    // 6. 通知应用层（但应用层暂停，收不到）
    NotifyApplication(SocketClosed);
}

// ⭐ 这整个过程完全在内核中自动完成
// ⭐ 不需要 Unity 应用层执行任何代码
```

**③ 为什么你在服务器看到的是"发消息时"？**

```
时间线分析：

服务器视角                          实际情况
────────────────────────────────────────────────────────
正常通信                            ✅ 双方都活跃
  ↓
(Unity 失焦，但服务器不知道)        ❌ Unity 暂停
  ↓
服务器发送数据                      ✅ 数据到达 Unity
  ↓                                 ❌ Unity 不处理
(没有响应)
  ↓
服务器 KeepAlive 定时器触发         ⏰ 30秒后
  ↓
服务器发送 Ping                     ✅ Ping 到达 Unity
  ↓                                 ❌ Unity 不回 Pong
等待 5 秒 (超时)
  ↓
服务器再发 Ping                     ✅ Ping 到达 Unity
  ↓                                 ❌ 还是不回 Pong
再等 5 秒 (超时)
  ↓
服务器判定：客户端挂了！            ⚠️ 决定关闭
  ↓
服务器发送 Close Frame              ✅ Close 到达 Unity
  ↓                                 ❌ Unity 不处理
服务器关闭 TCP 连接
  ↓
服务器发送 TCP FIN                  ✅ FIN 到达 Unity
  ↓                                 ✅ 操作系统自动四次挥手
服务器 Socket 关闭                  ✅ 双方 TCP 都关闭了
  ↓
服务器日志记录：                    📝 你看到的日志
  "连接关闭，原因：Ping 超时"

结论：
你看到的 Ping/Pong/Close 消息不是"发消息时触发的"
而是：
  1. 服务器发送的数据没有响应
  2. KeepAlive 检测到无响应
  3. 服务器主动关闭连接
  4. 你在服务器日志看到了这个过程
```

---

### **问题 2：Unity 焦点回归后自动重连的底层细节**

#### **完整的重连流程**

```
┌─────────────────────────────────────────────────────────────┐
│ Phase 1: Unity 恢复焦点                                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│ 用户切回 Unity 窗口                                          │
│   ↓                                                           │
│ Windows 消息队列                                             │
│   WM_ACTIVATE (窗口激活)                                     │
│   ↓                                                           │
│ Unity 引擎捕获                                               │
│   Application.isFocused = true                               │
│   ↓                                                           │
│ 触发生命周期回调                                             │
│   void OnApplicationPause(bool pauseStatus)                  │
│   {                                                           │
│       // pauseStatus = false (恢复运行)                      │
│       Debug.Log("Unity 恢复焦点");                           │
│   }                                                           │
│   ↓                                                           │
│ Unity 主线程恢复                                             │
│   ✅ Update() 开始执行                                       │
│   ✅ FixedUpdate() 开始执行                                  │
│   ✅ 协程 (Coroutine) 恢复                                   │
│   ✅ 异步任务 (async/await) 恢复                             │
│                                                               │
└─────────────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────────┐
│ Phase 2: WebSocket 接收线程恢复并发现连接已断开              │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│ WebSocket 接收循环恢复执行                                   │
│   ↓                                                           │
│ private async Task ReceiveLoop()                             │
│ {                                                             │
│     while (_isRunning)                                       │
│     {                                                         │
│         try                                                   │
│         {                                                     │
│             // ⭐ 这里恢复执行                                │
│             WebSocketReceiveResult result =                  │
│                 await _socket.ReceiveAsync(                  │
│                     buffer, CancellationToken.None);         │
│                                                               │
│             // ❌ 但此时 Socket 已经被操作系统关闭了！        │
│         }                                                     │
│         catch (WebSocketException ex)                        │
│         {                                                     │
│             // ⭐ 捕获异常：连接已关闭                        │
│             Debug.LogError($"WebSocket 异常: {ex.Message}"); │
│             // "An existing connection was forcibly          │
│             //  closed by the remote host"                   │
│                                                               │
│             // ⭐ 触发重连逻辑                                │
│             OnDisconnected?.Invoke();                        │
│             break;                                            │
│         }                                                     │
│     }                                                         │
│ }                                                             │
│                                                               │
└─────────────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────────┐
│ Phase 3: 自动重连逻辑触发                                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│ private void OnDisconnected()                                │
│ {                                                             │
│     Debug.Log("检测到断开连接，准备重连...");                │
│                                                               │
│     // 方式 1: 立即重连                                       │
│     Reconnect();                                             │
│                                                               │
│     // 方式 2: 延迟重连（更健壮）                             │
│     StartCoroutine(ReconnectWithDelay(3.0f));                │
│                                                               │
│     // 方式 3: 指数退避重连（生产环境推荐）                   │
│     StartCoroutine(ReconnectWithExponentialBackoff());       │
│ }                                                             │
│                                                               │
│ private IEnumerator ReconnectWithDelay(float delay)          │
│ {                                                             │
│     Debug.Log($"等待 {delay} 秒后重连...");                  │
│     yield return new WaitForSeconds(delay);                  │
│                                                               │
│     Debug.Log("开始重连...");                                │
│     await ConnectAsync();                                    │
│ }                                                             │
│                                                               │
└─────────────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────────┐
│ Phase 4: 建立新的 WebSocket 连接                             │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│ public async Task ConnectAsync()                             │
│ {                                                             │
│     // 1. 清理旧连接                                          │
│     if (_socket != null)                                     │
│     {                                                         │
│         _socket.Dispose();                                   │
│         _socket = null;                                      │
│     }                                                         │
│                                                               │
│     // 2. 创建新的 ClientWebSocket 实例                      │
│     _socket = new ClientWebSocket();                         │
│     _socket.Options.KeepAliveInterval = TimeSpan             │
│         .FromSeconds(30);                                    │
│                                                               │
│     // 3. 发起连接（完整的三次握手 + HTTP Upgrade）          │
│     try                                                       │
│     {                                                         │
│         Debug.Log("正在连接服务器...");                      │
│                                                               │
│         await _socket.ConnectAsync(                          │
│             new Uri("ws://server.com/ws"),                   │
│             CancellationToken.None);                         │
│                                                               │
│         Debug.Log("✅ 重连成功！");                          │
│                                                               │
│         // 4. 启动新的接收循环                               │
│         _isRunning = true;                                   │
│         _ = ReceiveLoop();                                   │
│                                                               │
│         // 5. 通知应用层                                      │
│         OnConnected?.Invoke();                               │
│     }                                                         │
│     catch (Exception ex)                                     │
│     {                                                         │
│         Debug.LogError($"❌ 重连失败: {ex.Message}");        │
│                                                               │
│         // 6. 失败后再次重试                                  │
│         StartCoroutine(ReconnectWithDelay(5.0f));            │
│     }                                                         │
│ }                                                             │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

#### **底层网络细节**

```
┌─────────────────────────────────────────────────────────────┐
│ 重连时的网络层活动                                           │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│ Step 1: DNS 查询 (如果使用域名)                              │
│   _socket.ConnectAsync(new Uri("ws://server.com/ws"))       │
│     ↓                                                         │
│   DNS 查询                                                   │
│     "server.com" → "93.184.216.34"                          │
│                                                               │
│ Step 2: TCP 三次握手                                         │
│   Unity                        服务器                        │
│     │                            │                           │
│     │  SYN (seq=1000)           │                           │
│     │─────────────────────────→ │  ① 请求连接               │
│     │                            │                           │
│     │  SYN-ACK (seq=2000,       │                           │
│     │           ack=1001)        │                           │
│     │ ←─────────────────────────│  ② 确认 + 请求           │
│     │                            │                           │
│     │  ACK (ack=2001)           │                           │
│     │─────────────────────────→ │  ③ 确认                   │
│     │                            │                           │
│     │   [TCP 连接建立]           │                           │
│                                                               │
│ Step 3: HTTP Upgrade 握手                                    │
│   Unity                        服务器                        │
│     │                            │                           │
│     │  GET /ws HTTP/1.1         │                           │
│     │  Upgrade: websocket       │                           │
│     │  Connection: Upgrade      │                           │
│     │  Sec-WebSocket-Key: ...   │                           │
│     │─────────────────────────→ │                           │
│     │                            │                           │
│     │  HTTP/1.1 101 Switching   │                           │
│     │  Upgrade: websocket       │                           │
│     │  Sec-WebSocket-Accept: ..│                           │
│     │ ←─────────────────────────│                           │
│     │                            │                           │
│     │   [WebSocket 连接建立]     │                           │
│                                                               │
│ Step 4: 开始正常通信                                         │
│   Unity                        服务器                        │
│     │                            │                           │
│     │  Text Frame: "重连成功"   │                           │
│     │─────────────────────────→ │                           │
│     │                            │                           │
│     │  Text Frame: "欢迎回来"   │                           │
│     │ ←─────────────────────────│                           │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---



