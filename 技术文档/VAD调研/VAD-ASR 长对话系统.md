---
创建时间: 2026-02-11 17:49
复习次数: 0
---

# VAD-ASR 长对话系统

  

## 架构总览

  

```

┌─────────────────────────────────────────────────────────────────────────┐

│                          Unity 主线程                                    │

│                                                                         │

│  ┌──────────┐    VAD事件     ┌──────────────────┐    录音控制           │

│  │ SileroVAD │──(后台线程)──→│ VADASRController  │──────────→┌────────┐ │

│  │ Manager   │   ↓           │  (状态机核心)      │           │ Record │ │

│  └──────────┘  MainThread    │                    │←──────────└────────┘ │

│                Dispatcher    └────────┬───────────┘               │      │

│                                       │ ASR完成事件               │      │

│                                       │ (MessageCenter)       首包/续包  │

│                                       │                      尾包(0x02) │

│                              ┌────────┴───────────┐               │      │

│                              │  XunFeiManager      │←─────────────┘      │

│                              │  (讯飞WebSocket ASR) │                     │

│                              └─────────────────────┘                     │

│                                  ↕ WebSocket (后台async线程)              │

│                              讯飞 STT 云服务                              │

└─────────────────────────────────────────────────────────────────────────┘

```

  

## 核心文件

  

| 文件 | 职责 |

|------|------|

| `Assets/AIChatTookit/VAD/VADASRController.cs` | 状态机核心，协调 VAD → 录音 → ASR 全流程 |

| `Assets/AIChatTookit/VAD/VADASRState.cs` | 状态枚举定义 |

| `Assets/AIChatTookit/Glue/Record.cs` | 麦克风录音、音频采集、首包/续包/尾包发送 |

| `Assets/AIChatTookit/ASR/XunFeiManager.cs` | 讯飞 WebSocket 连接、ASR 收发、结果解析 |

| `Assets/AIChatTookit/VAD/SileroVADManager.cs` | Silero ONNX 模型做 VAD 检测 |

| `Assets/AIChatTookit/VAD/MainThreadDispatcher.cs` | 后台线程→主线程调度器 |

| `Assets/AIChatTookit/VAD/AudioBufferManager.cs` | 线程安全音频环形缓冲区 |

  

## 状态机

  

```

┌──────┐

│ Idle │  ← StopLongDialogMode()

└──┬───┘

   │ StartLongDialogMode()

   ▼

┌────────────┐  VAD检测到人声   ┌───────────┐

│ WaitingVAD │────────────────→│ Recording │

└────────────┘                 └─────┬─────┘

   ▲                                 │

   │                    VAD检测到静音 → 启动800ms尾部静音协程

   │                                 │

   │                    800ms后仍静音 → StopRecordingByVAD(发尾包)

   │                                 │

   │                                 ▼

   │                        ┌────────────────┐

   │  CoustmonsSpeekOver    │ WaitingAsrDone │  ← 等待ASR返回最终结果

   │  (ASR完成事件)          │                │

   │◄───────────────────────┤  超时5s自动回退  │

   │                        └───────┬────────┘

   │                                │

   │    若 pendingVoiceStart=true    │ (WaitingAsrDone期间有新语音)

   │                                ▼

   │                        ┌───────────┐

   └────────────────────────│ Recording │  (直接开始新录音，不经过WaitingVAD)

                            └───────────┘

```

  

### 状态说明

  

| 状态 | 含义 | 允许的 VAD 事件响应 |

|------|------|---------------------|

| `Idle` | 未进入长对话 | 不响应 |

| `WaitingVAD` | 等待用户说话 | VStart → 进入 Recording |

| `Recording` | 录音中，音频持续发送给 ASR | VStart → 取消尾部静音计时器；VEnd → 启动尾部静音计时器 |

| `WaitingAsrDone` | 已发尾包，等 ASR 返回 | VStart → 标记 `pendingVoiceStart` |

| `Error` | 异常 | — |

  

## 单轮语音完整流程

  

```

时间轴 ─────────────────────────────────────────────────────────────→

  

用户:     [  说话中...  ]  [  静默  ]

VAD:   ───false───true─────────────false──────────────────────────────

                   │                 │

                   ▼                 ▼

Controller:   HandleLongDialogVAD(true)   HandleLongDialogVAD(false)

              ↓ MainThreadDispatcher       ↓ MainThreadDispatcher

              VStart()                     VEnd()

              ↓                            ↓

              状态→Recording               启动 TailSilenceCoroutine(800ms)

              ↓                                     │

              StartRecordingByVAD()                  │ 800ms后无人声

              ↓                                      ▼

Record:   GetAudoiDataFirst()              StopRecordingByVAD()

          ↓                                ↓

          SpeechToTextNew(首包)             发送尾包 0x02

          ↓                                状态→WaitingAsrDone

          GetAudoiDataNext(续包循环)                 │

          ↓                                          │

XunFei:   Connect→ASRSendFirstPack                   │

          STTSendMessageNew(续包×N)                   │

          STTReceiveMessage(持续接收)                  │

          ↓                                          │

          status==2 (最终结果)                        │

          ↓                                          │

          OnSelence() + Publish("CoustmonsSpeekOver") │

                              │                      │

                              ▼                      │

Controller:              OnAsrDone()                 │

                         状态→WaitingVAD ←───────────┘

```

  

## 关键机制

  

### 1. 尾部静音确认 (Tail Silence)

  

用户停止说话后不会立即停止录音，而是等待 `exitShort_config` (800ms)：

  

```

VEnd() 触发

  └→ 启动 LongDialogTailSilenceCoroutine(800ms)

       ├→ 800ms 内 VStart → 取消计时器，继续录音

       └→ 800ms 后无人声 → StopRecordingByVAD

```

  

防止用户说话时的自然停顿被误判为结束。

  

### 2. 录音会话版本号 (_recordingSessionVersion)

  

每次 VStart 进入 Recording 时递增，尾部静音协程携带版本号：

  

```

VStart → _recordingSessionVersion++ → 启动录音

VEnd   → LongDialogTailSilenceCoroutine(capturedVersion)

               └→ 若 capturedVersion != _recordingSessionVersion → 忽略

```

  

防止旧协程影响新的录音会话。

  

### 3. WaitingAsrDone + pendingVoiceStart (竞态保护)

  

解决核心竞态：用户在 ASR 还没返回时又开口说话。

  

```

状态=WaitingAsrDone 期间:

  VStart → _pendingVoiceStart = true （不立即开始录音）

  

ASR 完成 → OnAsrDone:

  若 pendingVoiceStart → 直接 Recording（跳过 WaitingVAD）

  否则               → WaitingVAD

```

  

### 4. WaitingAsrDone 超时保护

  

防止 ASR 异常（断网、异常）导致状态永久卡死：

  

```

进入 WaitingAsrDone → 启动 WaitingAsrDoneTimeoutCoroutine (5s)

  ├→ 5s 内收到 OnAsrDone → 取消超时协程，正常流转

  └→ 5s 超时 → 强制回退到 WaitingVAD（或带 pending 直接 Recording）

```

  

### 5. 线程安全

  

```

后台线程:                              主线程:

  SileroVAD.OnVoiceActivityChanged ──→ MainThreadDispatcher.Enqueue

  XunFei.STTReceiveMessage ──→         ├→ VStart() / VEnd()

    MessageCenter.Publish ──→          └→ OnAsrDone() (也通过 MainThreadDispatcher)

```

  

所有状态变更均在主线程执行。

  

## 配置参数 (Inspector)

  

| 参数 | 默认值 | 说明 |

|------|--------|------|

| `exitShort_config` | 800 | 尾部静音等待时间 (ms)，越大越不容易误切 |

| `voiceMean_config` | 300 | 最短有效语音时长 (ms)，短于此视为噪音 |

| `waitingAsrDoneTimeout` | 5.0 | WaitingAsrDone 超时 (s)，超时强制回退 |

  

## 日志追踪

  

所有关键日志统一使用 `[VAD-ASR]` 前缀，格式：

  

```

[VAD-ASR] [组件名] 内容

```

  

状态变化专用格式：

```

[VAD-ASR] [VADASRController] [ * 链路 ****** 1 ******* ] 状态变化为 : Recording 原因 : VAD检测到人声

```

  

搜索日志技巧：

- 搜 `链路` → 查看所有状态变化

- 搜 `尾部静音` → 查看静音确认流程

- 搜 `pendingVoiceStart` → 查看竞态触发情况

- 搜 `超时` → 查看 ASR 超时回退