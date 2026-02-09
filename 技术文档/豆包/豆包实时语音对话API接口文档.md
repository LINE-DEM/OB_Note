# 豆包实时语音对话API接口文档

  

## 📋 目录

  

1. [API概述](#api概述)

2. [认证与连接](#认证与连接)

3. [协议规范](#协议规范)

4. [消息类型](#消息类型)

5. [事件ID完整列表](#事件id完整列表)

6. [接口详细说明](#接口详细说明)

7. [数据结构](#数据结构)

8. [错误码](#错误码)

9. [代码示例](#代码示例)

10. [最佳实践](#最佳实践)

  

---

  

## 1. API概述

  

### 1.1 服务信息

  

| 项目 | 说明 |

|------|------|

| **服务名称** | 豆包端到端实时语音大模型 |

| **协议** | WebSocket (wss://) |

| **服务地址** | `wss://openspeech.bytedance.com/api/v3/realtime/dialogue` |

| **数据格式** | 二进制协议 (Header + Payload) |

| **编码** | UTF-8 (文本)，PCM (音频) |

  

### 1.2 功能特性

  

- ✅ **实时语音识别** (ASR)

- ✅ **大语言模型对话** (LLM)

- ✅ **语音合成** (TTS)

- ✅ **端到端低延迟** (<500ms)

- ✅ **多轮对话支持**

- ✅ **情感化语音**

- ✅ **可中断对话**

  

### 1.3 支持的音频格式

  

| 参数 | 输入 | 输出 |

|------|------|------|

| **采样率** | 16000 Hz | 24000 Hz |

| **声道** | 单声道 (Mono) | 单声道 (Mono) |

| **格式** | PCM S16LE | OGG Opus |

| **位深度** | 16-bit | - |

| **帧长** | 20ms (640字节) | 可变 |

  

---

  

## 2. 认证与连接

  

### 2.1 认证方式

  

使用**HTTP Header**进行认证：

  

```

X-Api-Resource-Id: volc.speech.dialog

X-Api-Access-Key: YOUR_ACCESS_KEY

X-Api-App-Key: YOUR_APP_KEY

X-Api-App-ID: YOUR_APP_ID

X-Api-Connect-Id: GENERATED_SESSION_ID

```

  

### 2.2 连接流程

  

```

1. 创建WebSocket连接

   ↓

2. 设置认证Header

   ↓

3. 调用 ws.Connect()

   ↓

4. 等待 OnOpen 事件

   ↓

5. 发送 StartConnection (事件ID=1)

   ↓

6. 接收 ConnectionStarted (事件ID=50)

   ↓

7. 连接成功

```

  

### 2.3 Unity实现示例

  

```csharp

using NativeWebSocket;

  

public class DouBaoConnector

{

    private WebSocket _webSocket;

  

    public async void Connect(string appId, string accessKey)

    {

        // 生成唯一会话ID

        string sessionId = Guid.NewGuid().ToString();

  

        // 设置认证Header

        var headers = new Dictionary<string, string>

        {

            { "X-Api-Resource-Id", "volc.speech.dialog" },

            { "X-Api-Access-Key", accessKey },

            { "X-Api-App-Key", "PlgvMymc7f3tQnJ6" },

            { "X-Api-App-ID", appId },

            { "X-Api-Connect-Id", sessionId }

        };

  

        // 创建WebSocket

        _webSocket = new WebSocket(

            "wss://openspeech.bytedance.com/api/v3/realtime/dialogue",

            headers

        );

  

        // 注册事件

        _webSocket.OnOpen += OnWebSocketOpen;

        _webSocket.OnMessage += OnWebSocketMessage;

        _webSocket.OnError += OnWebSocketError;

        _webSocket.OnClose += OnWebSocketClose;

  

        // 连接（非阻塞）

        _webSocket.Connect();

    }

  

    private void OnWebSocketOpen()

    {

        Debug.Log("WebSocket已连接");

        // 发送StartConnection消息

        SendStartConnection();

    }

}

```

  

---

  

## 3. 协议规范

  

### 3.1 二进制协议格式

  

每条消息由**Header + Payload**组成：

  

```

┌──────────────────────────────────────────────────────┐

│                    Message Format                     │

├──────────────────────────────────────────────────────┤

│  Header (4 bytes)                                    │

│  ┌─────────┬─────────┬──────────┬──────────────┐   │

│  │ Type(1) │ Event(1)│ Seq(1)   │ Len(1)       │   │

│  │ 1 byte  │ 1 byte  │ 1 byte   │ 1 byte       │   │

│  └─────────┴─────────┴──────────┴──────────────┘   │

├──────────────────────────────────────────────────────┤

│  Optional Fields (if Len > 0)                        │

│  ┌────────────────────────────────────────────┐     │

│  │  SessionID Length (1 byte)                 │     │

│  │  SessionID (variable)                      │     │

│  │  Query Length (2 bytes, big-endian)       │     │

│  │  Query ID (variable)                       │     │

│  └────────────────────────────────────────────┘     │

├──────────────────────────────────────────────────────┤

│  Payload (variable length)                           │

│  ┌────────────────────────────────────────────┐     │

│  │  JSON (FullClient/FullServer)              │     │

│  │  或                                         │     │

│  │  Binary Audio (AudioOnly)                  │     │

│  └────────────────────────────────────────────┘     │

└──────────────────────────────────────────────────────┘

```

  

### 3.2 Header字段说明

  

| 字段 | 字节 | 说明 |

|------|------|------|

| **Type** | 1 | 消息类型<br>0xB = FullClient<br>0xA = AudioOnlyClient<br>0x9 = FullServer<br>0x8 = AudioOnlyServer<br>0xF = Error |

| **Event** | 1 | 事件ID (见事件列表) |

| **Seq** | 1 | 序列号 (保留) |

| **Len** | 1 | 可选字段长度标志<br>bit0: 有SessionID<br>bit1: 有QueryID |

  

### 3.3 Protocol实现

  

```csharp

public static class RTDProtocol

{

    /// <summary>

    /// 消息类型

    /// </summary>

    public enum MessageType : byte

    {

        FullClient = 0x0B,        // 客户端JSON消息

        AudioOnlyClient = 0x0A,   // 客户端音频消息

        FullServer = 0x09,        // 服务端JSON消息

        AudioOnlyServer = 0x08,   // 服务端音频消息

        Error = 0x0F              // 错误消息

    }

  

    /// <summary>

    /// 封装消息

    /// </summary>

    public static byte[] Marshal(Message msg)

    {

        using (MemoryStream ms = new MemoryStream())

        {

            // 写入Header (4 bytes)

            ms.WriteByte((byte)msg.type);

            ms.WriteByte(msg.eventId);

            ms.WriteByte(msg.sequence);

            ms.WriteByte(msg.headerExtLen);

  

            // 写入可选字段

            if ((msg.headerExtLen & 0x01) != 0 && !string.IsNullOrEmpty(msg.sessionId))

            {

                byte[] sessionIdBytes = Encoding.UTF8.GetBytes(msg.sessionId);

                ms.WriteByte((byte)sessionIdBytes.Length);

                ms.Write(sessionIdBytes, 0, sessionIdBytes.Length);

            }

  

            if ((msg.headerExtLen & 0x02) != 0 && !string.IsNullOrEmpty(msg.queryId))

            {

                byte[] queryIdBytes = Encoding.UTF8.GetBytes(msg.queryId);

                ushort len = (ushort)queryIdBytes.Length;

                ms.WriteByte((byte)(len >> 8));

                ms.WriteByte((byte)(len & 0xFF));

                ms.Write(queryIdBytes, 0, queryIdBytes.Length);

            }

  

            // 写入Payload

            if (msg.payload != null && msg.payload.Length > 0)

            {

                ms.Write(msg.payload, 0, msg.payload.Length);

            }

  

            return ms.ToArray();

        }

    }

  

    /// <summary>

    /// 解析消息

    /// </summary>

    public static Message Unmarshal(byte[] data)

    {

        if (data == null || data.Length < 4)

            throw new Exception("消息数据不完整");

  

        Message msg = new Message

        {

            type = (MessageType)data[0],

            eventId = data[1],

            sequence = data[2],

            headerExtLen = data[3]

        };

  

        int offset = 4;

  

        // 读取SessionID

        if ((msg.headerExtLen & 0x01) != 0)

        {

            byte sessionIdLen = data[offset++];

            msg.sessionId = Encoding.UTF8.GetString(data, offset, sessionIdLen);

            offset += sessionIdLen;

        }

  

        // 读取QueryID

        if ((msg.headerExtLen & 0x02) != 0)

        {

            ushort queryIdLen = (ushort)((data[offset] << 8) | data[offset + 1]);

            offset += 2;

            msg.queryId = Encoding.UTF8.GetString(data, offset, queryIdLen);

            offset += queryIdLen;

        }

  

        // 读取Payload

        if (offset < data.Length)

        {

            int payloadLen = data.Length - offset;

            msg.payload = new byte[payloadLen];

            Array.Copy(data, offset, msg.payload, 0, payloadLen);

        }

  

        return msg;

    }

}

```

  

---

  

## 4. 消息类型

  

### 4.1 客户端消息类型

  

#### FullClient (0xB)

  

用于发送JSON格式的控制消息和文本消息。

  

**使用场景**：

- StartConnection

- StartSession

- EndSession

- ChatTextQuery (文本对话)

- SayHello (问候语)

  

#### AudioOnlyClient (0xA)

  

用于发送二进制音频数据。

  

**使用场景**：

- SendAudio (麦克风音频流)

- FinishAudio (音频结束标记)

  

### 4.2 服务端消息类型

  

#### FullServer (0x9)

  

服务器返回的JSON格式消息。

  

**包含内容**：

- 连接确认

- 会话状态

- ASR识别结果

- AI文本回复

- 事件通知

  

#### AudioOnlyServer (0x8)

  

服务器返回的二进制音频数据（TTS合成的语音）。

  

**格式**：OGG Opus编码

  

---

  

## 5. 事件ID完整列表

  

### 5.1 客户端事件 (Client → Server)

  

| 事件ID | 事件名称 | 说明 | 消息类型 |

|--------|---------|------|---------|

| **1** | StartConnection | 开始连接 | FullClient |

| **100** | StartSession | 开始会话 | FullClient |

| **101** | FinishSession | 结束会话 | FullClient |

| **200** | TaskRequest | 任务请求(保留) | FullClient |

| **201** | ChatTextQuery | 文本对话查询 | FullClient |

| **202** | SayHello | 发送问候语 | FullClient |

| **300** | SendAudio | 发送音频数据 | AudioOnlyClient |

| **301** | FinishAudio | 音频发送结束 | AudioOnlyClient |

  

### 5.2 服务端事件 (Server → Client)

  

| 事件ID | 事件名称 | 说明 | 消息类型 |

|--------|---------|------|---------|

| **50** | ConnectionStarted | 连接已建立 | FullServer |

| **150** | SessionStarted | 会话已开始 | FullServer |

| **151** | SessionFinished | 会话已结束 | FullServer |

| **250** | TaskStarted | 任务开始 | FullServer |

| **251** | TaskFinished | 任务结束 | FullServer |

| **350** | SpeechStart | 用户开始说话 | FullServer |

| **351** | SpeechEnd | 用户停止说话 | FullServer |

| **450** | ASRStart | ASR识别开始 | FullServer |

| **451** | ASRResponse | ASR识别结果 | FullServer |

| **452** | ASRFinish | ASR识别完成 | FullServer |

| **500** | ChatStart | AI回复开始 | FullServer |

| **501** | ChatResponse | AI回复内容(流式) | FullServer |

| **502** | ChatFinish | AI回复结束 | FullServer |

| **600** | TTSStart | TTS合成开始 | FullServer |

| **601** | TTSResponse | TTS音频数据 | AudioOnlyServer |

| **602** | TTSFinish | TTS合成结束 | FullServer |

| **6000** | Error | 错误消息 | Error (0xF) |

  

---

  

## 6. 接口详细说明

  

### 6.1 连接管理

  

#### 6.1.1 StartConnection (事件ID=1)

  

**用途**：初始化WebSocket连接

  

**发送时机**：WebSocket OnOpen后立即发送

  

**消息格式**：

```csharp

MessageType: FullClient (0xB)

EventID: 1

Payload: {} (空JSON对象)

```

  

**代码示例**：

```csharp

public static byte[] CreateStartConnectionMessage()

{

    Message msg = new Message

    {

        type = MessageType.FullClient,

        eventId = 1,  // StartConnection

        sequence = 0,

        headerExtLen = 0,

        payload = Encoding.UTF8.GetBytes("{}")

    };

    return Marshal(msg);

}

```

  

**服务器响应**：

- 事件ID：50 (ConnectionStarted)

- 表示连接已确认

  

---

  

#### 6.1.2 ConnectionStarted (事件ID=50)

  

**用途**：服务器确认连接成功

  

**接收示例**：

```json

{

    "code": 0,

    "message": "success",

    "dialog_id": "dialog_20260109123456"

}

```

  

**字段说明**：

- `code`: 状态码，0表示成功

- `message`: 状态消息

- `dialog_id`: 对话ID

  

---

  

### 6.2 会话管理

  

#### 6.2.1 StartSession (事件ID=100)

  

**用途**：开始一个新的对话会话

  

**发送时机**：连接成功后

  

**Payload结构**：

```json

{

    "app": {

        "appid": "YOUR_APP_ID"

    },

    "user": {

        "uid": "user_12345"

    },

    "audio": {

        "format": "pcm",

        "sample_rate": 16000,

        "channel": 1,

        "bits": 16,

        "encoding": "pcm_s16le"

    },

    "request": {

        "reqid": "req_uuid_12345",

        "input_mode": "audio",

        "model_version": "O",

        "bot_name": "豆包",

        "system_role": "你是一个热情友好的AI助手",

        "speaking_style": "简洁明了，语速适中",

        "voice": {

            "speaker": "zh_female_vv_jupiter_bigtts",

            "sample_rate": 24000,

            "format": "ogg_opus"

        }

    }

}

```

  

**字段详解**：

  

| 字段路径 | 类型 | 必填 | 说明 |

|---------|------|------|------|

| `app.appid` | string | ✅ | 应用ID |

| `user.uid` | string | ✅ | 用户ID |

| `audio.format` | string | ✅ | 输入音频格式，固定"pcm" |

| `audio.sample_rate` | int | ✅ | 输入采样率，16000 |

| `audio.channel` | int | ✅ | 声道数，1 |

| `audio.bits` | int | ✅ | 位深度，16 |

| `audio.encoding` | string | ✅ | 编码方式，"pcm_s16le" |

| `request.input_mode` | string | ✅ | 输入模式：audio / text / audio_file / keep_alive |

| `request.model_version` | string | ✅ | 模型版本：O / SC / O2_0 / SC2_0 |

| `request.bot_name` | string | ❌ | 机器人名称 |

| `request.system_role` | string | ❌ | 系统角色定义 |

| `request.speaking_style` | string | ❌ | 说话风格 |

| `request.voice.speaker` | string | ✅ | 发音人ID |

| `request.voice.sample_rate` | int | ✅ | 输出采样率，24000 |

| `request.voice.format` | string | ✅ | 输出格式，"ogg_opus" |

  

**代码示例**：

```csharp

public static byte[] CreateStartSessionMessage(string sessionId, string jsonPayload)

{

    Message msg = new Message

    {

        type = MessageType.FullClient,

        eventId = 100,  // StartSession

        sequence = 0,

        headerExtLen = 0x01,  // 包含SessionID

        sessionId = sessionId,

        payload = Encoding.UTF8.GetBytes(jsonPayload)

    };

    return Marshal(msg);

}

```

  

**服务器响应**：

- 事件ID：150 (SessionStarted)

  

---

  

#### 6.2.2 SessionStarted (事件ID=150)

  

**用途**：服务器确认会话已开始

  

**接收示例**：

```json

{

    "code": 0,

    "message": "session started",

    "dialog_id": "dialog_20260109123456",

    "session_id": "session_uuid"

}

```

  

**处理逻辑**：

```csharp

private void OnSessionStarted(string jsonPayload)

{

    var data = JsonUtility.FromJson<SessionStartedData>(jsonPayload);

    _dialogId = data.dialog_id;

    _isSessionActive = true;

  

    Debug.Log($"会话已开始: {_dialogId}");

    OnSessionStarted?.Invoke(_dialogId);

  

    // 现在可以开始发送音频或文本

}

```

  

---

  

#### 6.2.3 FinishSession (事件ID=101)

  

**用途**：结束当前会话

  

**发送时机**：对话结束时

  

**Payload**：

```json

{

    "dialog_id": "dialog_20260109123456"

}

```

  

**代码示例**：

```csharp

public static byte[] CreateFinishSessionMessage(string sessionId)

{

    var payload = new { dialog_id = sessionId };

    string json = JsonUtility.ToJson(payload);

  

    Message msg = new Message

    {

        type = MessageType.FullClient,

        eventId = 101,  // FinishSession

        sequence = 0,

        headerExtLen = 0x01,

        sessionId = sessionId,

        payload = Encoding.UTF8.GetBytes(json)

    };

    return Marshal(msg);

}

```

  

---

  

### 6.3 音频处理

  

#### 6.3.1 SendAudio (事件ID=300)

  

**用途**：发送音频数据到服务器

  

**发送频率**：每20ms一次（640字节/帧）

  

**消息格式**：

```

MessageType: AudioOnlyClient (0xA)

EventID: 300

SessionID: 必填

QueryID: 选填（每次用户说话生成一个）

Payload: PCM音频数据 (640字节)

```

  

**代码示例**：

```csharp

public static byte[] CreateAudioMessage(string sessionId, byte[] audioData)

{

    Message msg = new Message

    {

        type = MessageType.AudioOnlyClient,

        eventId = 300,  // SendAudio

        sequence = 0,

        headerExtLen = 0x01,  // 包含SessionID

        sessionId = sessionId,

        payload = audioData

    };

    return Marshal(msg);

}

  

// 使用示例

private void SendMicrophoneData()

{

    byte[] audioChunk = GetMicrophoneData(); // 640字节

    byte[] message = RTDProtocol.CreateAudioMessage(_sessionId, audioChunk);

    _webSocket.Send(message);

}

```

  

**音频参数**：

- 采样率：16000 Hz

- 格式：PCM S16LE (小端序16位)

- 声道：单声道

- 帧长：20ms = 640字节 (16000 * 2 * 0.02)

  

---

  

#### 6.3.2 FinishAudio (事件ID=301)

  

**用途**：标记音频输入结束（可选）

  

**发送时机**：用户停止说话时

  

**代码示例**：

```csharp

public static byte[] CreateFinishAudioMessage(string sessionId, string queryId)

{

    Message msg = new Message

    {

        type = MessageType.AudioOnlyClient,

        eventId = 301,  // FinishAudio

        sequence = 0,

        headerExtLen = 0x03,  // SessionID + QueryID

        sessionId = sessionId,

        queryId = queryId,

        payload = new byte[0]

    };

    return Marshal(msg);

}

```

  

---

  

#### 6.3.3 TTSResponse (事件ID=601)

  

**用途**：接收服务器返回的TTS音频

  

**消息格式**：

```

MessageType: AudioOnlyServer (0x8)

EventID: 601

Payload: OGG Opus音频数据

```

  

**处理示例**：

```csharp

private void OnTTSAudioReceived(byte[] audioData)

{

    Debug.Log($"收到TTS音频: {audioData.Length} 字节");

  

    // 保存或播放音频

    _receivedAudioData.AddRange(audioData);

  

    // TODO: 解码Opus音频并播放

    // AudioClip clip = DecodeOpus(audioData);

    // _audioSource.PlayOneShot(clip);

  

    OnAudioReceived?.Invoke(audioData);

}

```

  

---

  

### 6.4 文本处理

  

#### 6.4.1 ChatTextQuery (事件ID=201)

  

**用途**：发送文本消息进行对话

  

**Payload结构**：

```json

{

    "dialog_id": "dialog_20260109123456",

    "query": {

        "text": "今天天气怎么样？"

    }

}

```

  

**代码示例**：

```csharp

public static byte[] CreateChatTextQueryMessage(string sessionId, string text)

{

    var payload = new

    {

        dialog_id = sessionId,

        query = new { text = text }

    };

    string json = JsonUtility.ToJson(payload);

  

    Message msg = new Message

    {

        type = MessageType.FullClient,

        eventId = 201,  // ChatTextQuery

        sequence = 0,

        headerExtLen = 0x01,

        sessionId = sessionId,

        payload = Encoding.UTF8.GetBytes(json)

    };

    return Marshal(msg);

}

```

  

---

  

#### 6.4.2 SayHello (事件ID=202)

  

**用途**：发送初始问候语

  

**Payload结构**：

```json

{

    "dialog_id": "dialog_20260109123456",

    "text": "你好，我是你的AI助手"

}

```

  

**代码示例**：

```csharp

public static byte[] CreateSayHelloMessage(string sessionId, string greeting)

{

    var payload = new

    {

        dialog_id = sessionId,

        text = greeting

    };

    string json = JsonUtility.ToJson(payload);

  

    Message msg = new Message

    {

        type = MessageType.FullClient,

        eventId = 202,  // SayHello

        sequence = 0,

        headerExtLen = 0x01,

        sessionId = sessionId,

        payload = Encoding.UTF8.GetBytes(json)

    };

    return Marshal(msg);

}

```

  

---

  

### 6.5 实时反馈

  

#### 6.5.1 ASRResponse (事件ID=451)

  

**用途**：实时语音识别结果

  

**Payload结构**：

```json

{

    "code": 0,

    "message": "success",

    "query_id": "query_uuid",

    "result": {

        "text": "今天天气怎么样",

        "is_final": false,

        "confidence": 0.95

    }

}

```

  

**字段说明**：

- `is_final`: false=临时结果，true=最终结果

- `confidence`: 识别置信度 (0-1)

  

**处理示例**：

```csharp

private void OnASRResponse(string jsonPayload)

{

    var data = JsonUtility.FromJson<ASRResponseData>(jsonPayload);

  

    Debug.Log($"ASR识别: {data.result.text} (final={data.result.is_final})");

  

    OnASRTextReceived?.Invoke(data.result.text, !data.result.is_final);

  

    if (data.result.is_final)

    {

        // 最终结果，可以显示在UI上

        Debug.Log($"最终识别结果: {data.result.text}");

    }

}

```

  

---

  

#### 6.5.2 ChatResponse (事件ID=501)

  

**用途**：AI回复内容（流式返回）

  

**Payload结构**：

```json

{

    "code": 0,

    "message": "success",

    "query_id": "query_uuid",

    "result": {

        "text": "今天天气很好，阳光明媚",

        "is_final": false

    }

}

```

  

**特点**：

- 流式返回：服务器会多次发送此事件，逐步返回完整回复

- `is_final=true` 表示回复结束

  

**处理示例**：

```csharp

private string _currentChatText = "";

  

private void OnChatResponse(string jsonPayload)

{

    var data = JsonUtility.FromJson<ChatResponseData>(jsonPayload);

  

    if (data.result.is_final)

    {

        // 最终回复

        _currentChatText = data.result.text;

        OnChatTextReceived?.Invoke(_currentChatText, false);

  

        Debug.Log($"AI最终回复: {_currentChatText}");

    }

    else

    {

        // 流式中间结果

        _currentChatText = data.result.text;

        OnChatTextReceived?.Invoke(_currentChatText, true);

  

        // 实时显示在UI上（打字机效果）

    }

}

```

  

---

  

#### 6.5.3 SpeechStart / SpeechEnd (事件ID=350/351)

  

**用途**：检测用户说话开始/结束

  

**SpeechStart (350) Payload**：

```json

{

    "query_id": "query_uuid_12345",

    "timestamp": 1673456789

}

```

  

**SpeechEnd (351) Payload**：

```json

{

    "query_id": "query_uuid_12345",

    "timestamp": 1673456792

}

```

  

**处理示例**：

```csharp

private void OnSpeechStart(string jsonPayload)

{

    var data = JsonUtility.FromJson<SpeechEventData>(jsonPayload);

    _currentQueryId = data.query_id;

  

    Debug.Log($"用户开始说话: {_currentQueryId}");

    OnUserSpeakingStart?.Invoke(_currentQueryId);

  

    // UI显示"正在听..."

}

  

private void OnSpeechEnd(string jsonPayload)

{

    Debug.Log("用户停止说话");

    OnUserSpeakingEnd?.Invoke();

  

    // UI显示"识别中..."

}

```

  

---

  

#### 6.5.4 TTSStart / TTSFinish (事件ID=600/602)

  

**用途**：TTS合成开始/结束通知

  

**TTSStart (600) Payload**：

```json

{

    "query_id": "query_uuid",

    "text": "今天天气很好",

    "speaker": "zh_female_vv_jupiter_bigtts"

}

```

  

**TTSFinish (602) Payload**：

```json

{

    "query_id": "query_uuid",

    "status": "completed"

}

```

  

**处理示例**：

```csharp

private void OnTTSStart(string jsonPayload)

{

    var data = JsonUtility.FromJson<TTSStartData>(jsonPayload);

  

    Debug.Log($"TTS开始: {data.text}");

    OnTTSStart?.Invoke(data.speaker, data.text);

  

    // 触发角色动画

    // 开始口型同步

}

  

private void OnTTSFinish(string jsonPayload)

{

    Debug.Log("TTS结束");

    OnTTSEnd?.Invoke();

  

    // 停止角色动画

    // 停止口型同步

}

```

  

---

  

### 6.6 错误处理

  

#### Error (事件ID=6000)

  

**Payload结构**：

```json

{

    "code": 40001,

    "message": "认证失败：无效的Access Key",

    "details": {

        "error_type": "AUTH_ERROR",

        "timestamp": 1673456789

    }

}

```

  

**常见错误码**：

  

| 错误码 | 说明 | 处理建议 |

|--------|------|---------|

| 40000 | 请求参数错误 | 检查Payload格式 |

| 40001 | 认证失败 | 检查API密钥 |

| 40002 | 权限不足 | 检查服务是否开通 |

| 40003 | 配额超限 | 等待配额恢复或升级 |

| 50000 | 服务器内部错误 | 重试或联系技术支持 |

| 50001 | 服务暂时不可用 | 稍后重试 |

  

**处理示例**：

```csharp

private void OnErrorMessage(string jsonPayload)

{

    var error = JsonUtility.FromJson<ErrorData>(jsonPayload);

  

    Debug.LogError($"错误 [{error.code}]: {error.message}");

    OnError?.Invoke(error.message, error.code);

  

    // 根据错误码处理

    switch (error.code)

    {

        case 40001: // 认证失败

            Debug.LogError("API认证失败，请检查密钥配置");

            break;

  

        case 40003: // 配额超限

            Debug.LogWarning("API配额已用完");

            break;

  

        case 50000: // 服务器错误

            Debug.LogError("服务器错误，尝试重连...");

            RetryConnection();

            break;

    }

}

```

  

---

  

## 7. 数据结构

  

### 7.1 C# 数据类定义

  

```csharp

// 会话开始响应

[Serializable]

public class SessionStartedData

{

    public int code;

    public string message;

    public string dialog_id;

    public string session_id;

}

  

// ASR识别结果

[Serializable]

public class ASRResponseData

{

    public int code;

    public string message;

    public string query_id;

    public ASRResult result;

}

  

[Serializable]

public class ASRResult

{

    public string text;

    public bool is_final;

    public float confidence;

}

  

// Chat回复

[Serializable]

public class ChatResponseData

{

    public int code;

    public string message;

    public string query_id;

    public ChatResult result;

}

  

[Serializable]

public class ChatResult

{

    public string text;

    public bool is_final;

}

  

// 说话事件

[Serializable]

public class SpeechEventData

{

    public string query_id;

    public long timestamp;

}

  

// TTS开始

[Serializable]

public class TTSStartData

{

    public string query_id;

    public string text;

    public string speaker;

}

  

// 错误数据

[Serializable]

public class ErrorData

{

    public int code;

    public string message;

    public ErrorDetails details;

}

  

[Serializable]

public class ErrorDetails

{

    public string error_type;

    public long timestamp;

}

```

  

---

  

## 8. 错误码

  

### 8.1 客户端错误 (4xxxx)

  

| 错误码 | 错误类型 | 说明 |

|--------|---------|------|

| 40000 | INVALID_PARAMETER | 请求参数格式错误 |

| 40001 | AUTH_FAILED | API认证失败 |

| 40002 | PERMISSION_DENIED | 权限不足 |

| 40003 | QUOTA_EXCEEDED | 配额超限 |

| 40004 | RATE_LIMIT | 请求频率超限 |

| 40005 | INVALID_SESSION | 无效的会话ID |

| 40006 | SESSION_EXPIRED | 会话已过期 |

  

### 8.2 服务端错误 (5xxxx)

  

| 错误码 | 错误类型 | 说明 |

|--------|---------|------|

| 50000 | INTERNAL_ERROR | 服务器内部错误 |

| 50001 | SERVICE_UNAVAILABLE | 服务暂时不可用 |

| 50002 | TIMEOUT | 请求超时 |

| 50003 | ASR_ERROR | 语音识别失败 |

| 50004 | LLM_ERROR | 大模型响应失败 |

| 50005 | TTS_ERROR | 语音合成失败 |

  

---

  

## 9. 代码示例

  

### 9.1 完整连接流程

  

```csharp

using UnityEngine;

using NativeWebSocket;

using System;

using System.Text;

  

public class DouBaoAPIExample : MonoBehaviour

{

    private WebSocket _webSocket;

    private string _sessionId;

    private bool _isConnected;

    private bool _isSessionActive;

  

    // 配置

    private const string APP_ID = "YOUR_APP_ID";

    private const string ACCESS_KEY = "YOUR_ACCESS_KEY";

  

    async void Start()

    {

        // 1. 连接

        await ConnectToDouBao();

    }

  

    void Update()

    {

        // NativeWebSocket需要在Update中调度消息

        _webSocket?.DispatchMessageQueue();

    }

  

    private async UniTask ConnectToDouBao()

    {

        // 生成会话ID

        _sessionId = Guid.NewGuid().ToString();

  

        // 设置Header

        var headers = new Dictionary<string, string>

        {

            { "X-Api-Resource-Id", "volc.speech.dialog" },

            { "X-Api-Access-Key", ACCESS_KEY },

            { "X-Api-App-Key", "PlgvMymc7f3tQnJ6" },

            { "X-Api-App-ID", APP_ID },

            { "X-Api-Connect-Id", _sessionId }

        };

  

        // 创建WebSocket

        _webSocket = new WebSocket(

            "wss://openspeech.bytedance.com/api/v3/realtime/dialogue",

            headers

        );

  

        // 注册回调

        _webSocket.OnOpen += OnOpen;

        _webSocket.OnMessage += OnMessage;

        _webSocket.OnError += OnError;

        _webSocket.OnClose += OnClose;

  

        // 连接

        await _webSocket.Connect();

    }

  

    private void OnOpen()

    {

        Debug.Log("✅ WebSocket已连接");

  

        // 发送StartConnection

        byte[] msg = RTDProtocol.CreateStartConnectionMessage();

        _webSocket.Send(msg);

    }

  

    private void OnMessage(byte[] data)

    {

        // 解析消息

        var message = RTDProtocol.Unmarshal(data);

  

        // 根据事件ID处理

        switch (message.eventId)

        {

            case 50: // ConnectionStarted

                OnConnectionStarted(message);

                break;

  

            case 150: // SessionStarted

                OnSessionStarted(message);

                break;

  

            case 451: // ASRResponse

                OnASRResponse(message);

                break;

  

            case 501: // ChatResponse

                OnChatResponse(message);

                break;

  

            case 601: // TTSResponse (音频)

                OnTTSAudio(message);

                break;

        }

    }

  

    private void OnConnectionStarted(RTDProtocol.Message message)

    {

        Debug.Log("✅ 连接已确认，开始会话");

        _isConnected = true;

  

        // 开始会话

        StartSession();

    }

  

    private void StartSession()

    {

        // 构建StartSession Payload

        var payload = new

        {

            app = new { appid = APP_ID },

            user = new { uid = "user123" },

            audio = new

            {

                format = "pcm",

                sample_rate = 16000,

                channel = 1,

                bits = 16,

                encoding = "pcm_s16le"

            },

            request = new

            {

                reqid = Guid.NewGuid().ToString(),

                input_mode = "audio",

                model_version = "O",

                bot_name = "豆包",

                voice = new

                {

                    speaker = "zh_female_vv_jupiter_bigtts",

                    sample_rate = 24000,

                    format = "ogg_opus"

                }

            }

        };

  

        string json = JsonUtility.ToJson(payload);

        byte[] msg = RTDProtocol.CreateStartSessionMessage(_sessionId, json);

        _webSocket.Send(msg);

    }

  

    private void OnSessionStarted(RTDProtocol.Message message)

    {

        Debug.Log("✅ 会话已开始");

        _isSessionActive = true;

  

        // 现在可以启动麦克风

        StartMicrophone();

    }

  

    private void StartMicrophone()

    {

        // 启动麦克风录音

        AudioClip micClip = Microphone.Start(null, true, 1, 16000);

        // 定期发送音频数据...

    }

  

    private void OnASRResponse(RTDProtocol.Message message)

    {

        string json = Encoding.UTF8.GetString(message.payload);

        var data = JsonUtility.FromJson<ASRResponseData>(json);

  

        Debug.Log($"🎤 ASR: {data.result.text}");

    }

  

    private void OnChatResponse(RTDProtocol.Message message)

    {

        string json = Encoding.UTF8.GetString(message.payload);

        var data = JsonUtility.FromJson<ChatResponseData>(json);

  

        Debug.Log($"💬 AI: {data.result.text}");

    }

  

    private void OnTTSAudio(RTDProtocol.Message message)

    {

        byte[] audioData = message.payload;

        Debug.Log($"🔊 收到TTS音频: {audioData.Length} 字节");

  

        // TODO: 解码Opus并播放

    }

  

    private void OnError(string error)

    {

        Debug.LogError($"❌ WebSocket错误: {error}");

    }

  

    private void OnClose(WebSocketCloseCode closeCode)

    {

        Debug.Log($"⛔ WebSocket已关闭: {closeCode}");

        _isConnected = false;

        _isSessionActive = false;

    }

  

    void OnDestroy()

    {

        _webSocket?.Close();

    }

}

```

  

---

  

## 10. 最佳实践

  

### 10.1 连接管理

  

```csharp

// ✅ 推荐：非阻塞连接

_webSocket.Connect(); // 不要await

await UniTask.Yield(); // 让Update循环运行

  

// ❌ 避免：阻塞连接

await _webSocket.Connect(); // 会导致死锁

```

  

### 10.2 音频发送

  

```csharp

// ✅ 推荐：固定20ms间隔发送

private IEnumerator SendAudioCoroutine()

{

    while (_isRecording)

    {

        byte[] audioChunk = GetMicrophoneData(640); // 640字节

        SendAudioData(audioChunk);

        yield return new WaitForSeconds(0.02f); // 20ms

    }

}

  

// ❌ 避免：不规则发送或过大的块

SendAudioData(bigChunk); // 太大会导致延迟

```

  

### 10.3 错误处理

  

```csharp

// ✅ 推荐：完善的错误处理

try

{

    await ConnectAsync();

}

catch (Exception ex)

{

    Debug.LogError($"连接失败: {ex.Message}");

    // 显示用户友好的错误提示

    // 记录日志

    // 触发重连机制

}

  

// ❌ 避免：忽略错误

await ConnectAsync(); // 可能会崩溃

```

  

### 10.4 资源清理

  

```csharp

// ✅ 推荐：及时清理资源

void OnDestroy()

{

    StopMicrophone();

    _webSocket?.Close();

    ClearAudioBuffers();

}

  

// ❌ 避免：忘记清理

void OnDestroy() { } // WebSocket可能没关闭

```

  

### 10.5 状态管理

  

```csharp

// ✅ 推荐：严格的状态检查

public void SendText(string text)

{

    if (!_isConnected)

    {

        Debug.LogWarning("未连接，无法发送");

        return;

    }

  

    if (!_isSessionActive)

    {

        Debug.LogWarning("会话未活跃，无法发送");

        return;

    }

  

    // 发送消息...

}

  

// ❌ 避免：不检查状态

public void SendText(string text)

{

    // 直接发送，可能会失败

    _webSocket.Send(data);

}

```

  

---

  

## 11. 常见问题FAQ

  

### Q1: 为什么音频没有被识别？

  

**A**: 检查以下几点：

1. 采样率是否为16000 Hz

2. 格式是否为PCM S16LE

3. 每帧大小是否为640字节 (20ms)

4. 会话是否已开始 (SessionStarted)

  

### Q2: TTS音频如何播放？

  

**A**: 豆包返回的是OGG Opus格式，需要：

1. 集成Opus解码库（如：opus-tools）

2. 解码为PCM数据

3. 创建AudioClip并播放

  

```csharp

// 示例（需要Opus解码器）

byte[] pcmData = OpusDecoder.Decode(oggOpusData);

AudioClip clip = AudioClip.Create("TTS", pcmData.Length / 2, 1, 24000, false);

clip.SetData(ConvertBytesToFloat(pcmData), 0);

audioSource.PlayOneShot(clip);

```

  

### Q3: 如何降低延迟？

  

**A**:

1. 使用音频模式（不要文本模式）

2. 减小音频缓冲区大小

3. 确保网络稳定

4. 使用低延迟的音频设置

  

```csharp

var config = AudioSettings.GetConfiguration();

config.dspBufferSize = 512; // 降低缓冲

AudioSettings.Reset(config);

```

  

### Q4: 如何实现多轮对话？

  

**A**: 保持会话活跃即可：

1. 一次StartSession

2. 多次发送音频/文本

3. 服务器会自动维护上下文

4. 结束时调用FinishSession

  

### Q5: 成本如何计算？

  

**A**:

- 按**实际语音通话时长**计费

- 从SessionStarted到SessionFinished

- 即使用户没说话，只要会话活跃就计费

- 建议实现超时自动结束机制

  

---

  

## 12. 附录

  

### 12.1 发音人列表

  

| ID | 名称 | 性别 | 风格 |

|----|------|------|------|

| zh_female_vv_jupiter_bigtts | 木星女声 | 女 | 温柔亲切 |

| zh_male_vv_mars_bigtts | 火星男声 | 男 | 沉稳大气 |

| zh_female_vv_venus_bigtts | 金星女声 | 女 | 活泼可爱 |

| zh_male_vv_saturn_bigtts | 土星男声 | 男 | 专业严肃 |

  

### 12.2 模型版本

  

| 版本 | 说明 | 适用场景 |

|------|------|---------|

| O | 标准版 | 通用对话 |

| SC | 声音复刻版 | 需要特定音色 |

| O2_0 | 增强版 (1.2.1.0) | 更自然的对话 |

| SC2_0 | 声音复刻增强版 | 高质量音色 |

  

### 12.3 输入模式

  

| 模式 | 说明 |

|------|------|

| audio | 实时音频输入（推荐） |

| text | 纯文本对话 |

| audio_file | 音频文件处理 |

| keep_alive | 保活模式（麦克风静音） |

  

---

  

**文档版本**：v1.0

**最后更新**：2026-01-09

**官方文档**：https://www.volcengine.com/docs/6561/1221097

**技术支持**：火山引擎控制台工单系统