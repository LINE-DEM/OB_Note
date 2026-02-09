# Unity实时语音对话API使用手册

  

## 目录

1. [简介](#简介)

2. [功能特性](#功能特性)

3. [系统要求](#系统要求)

4. [快速开始](#快速开始)

5. [API参考](#api参考)

6. [高级用法](#高级用法)

7. [常见问题](#常见问题)

8. [错误处理](#错误处理)

9. [最佳实践](#最佳实践)

  

---

  

## 简介

  

火山引擎实时语音对话API（Volcengine Realtime Dialog API）是一个端到端的实时语音交互系统，支持：

- 低延迟语音到语音对话

- 实时语音识别（ASR）

- 实时语音合成（TTS）

- 多模式输入（语音、文本、音频文件）

- 流式交互

  

本Unity实现提供了完整的C# API封装，方便在Unity项目中集成实时语音对话功能。

  

---

  

## 功能特性

  

### 核心功能

- ✅ WebSocket连接管理

- ✅ 实时麦克风录音

- ✅ 实时音频播放

- ✅ 语音识别（ASR）

- ✅ 语音合成（TTS）

- ✅ 文本输入/输出

- ✅ 会话管理

- ✅ 事件驱动架构

  

### 支持的模型版本

- **O版本** - 支持精品音色（vv、xiaohe、yunzhou、xiaotian）

- **SC版本** - 支持声音复刻

- **O2.0版本** - O版本升级，增强唱歌能力

- **SC2.0版本** - SC版本升级，增强角色演绎

  

### 支持的输入模式

- **Audio** - 麦克风实时输入

- **AudioFile** - 音频文件输入

- **Text** - 纯文本输入

- **KeepAlive** - 麦克风静音保活模式

  

---

  

## 系统要求

  

### Unity版本

- Unity 2021.3 LTS或更高版本

- 支持.NET Standard 2.1

  

### 依赖项

- **UniTask** - 用于异步操作（已在项目中）

- **WebSocket库**（需要额外安装）:

  - 推荐：NativeWebSocket

  - 或：WebSocketSharp

  - 或：Unity WebSocket Package

  

### 平台支持

- ✅ Windows（编辑器和独立版本）

- ✅ macOS（编辑器和独立版本）

- ✅ Android（需要麦克风权限）

- ✅ iOS（需要麦克风权限）

- ⚠️ WebGL（WebSocket支持有限）

  

---

  

## 快速开始

  

### 1. 准备工作

  

#### 获取API凭证

1. 访问[火山引擎控制台](https://console.volcengine.com/)

2. 开通"豆包端到端实时语音大模型"服务

3. 获取以下信息：

   - `API App ID`

   - `API Access Key`

  

#### 安装WebSocket库（以NativeWebSocket为例）

```bash

# 使用Unity Package Manager安装

https://github.com/endel/NativeWebSocket.git#upm

```

  

### 2. 基础使用示例

  

#### 步骤1：创建配置

在Unity编辑器中创建RTDConfig配置：

  

```csharp

using VolcEngine.RealtimeDialog;

using UnityEngine;

  

[CreateAssetMenu(fileName = "RTDConfig", menuName = "RealtimeDialog/Config")]

public class MyRTDConfig : ScriptableObject

{

    public RTDConfig config;

}

```

  

#### 步骤2：设置配置

在Inspector中填写配置：

  

```yaml

wsUrl: "wss://openspeech.bytedance.com/api/v3/realtime/dialogue"

apiAppId: "YOUR_APP_ID"

apiAccessKey: "YOUR_ACCESS_KEY"

inputSampleRate: 16000

outputSampleRate: 24000

pcmFormat: Pcm_S16LE

defaultSpeaker: "zh_female_vv_jupiter_bigtts"

inputMode: Audio

modelVersion: O

botName: "豆包"

enableVerboseLogging: true

```

  

#### 步骤3：使用API

  

```csharp

using UnityEngine;

using VolcEngine.RealtimeDialog;

  

public class SimpleExample : MonoBehaviour

{

    [SerializeField] private RTDConfig config;

    private IRealtimeDialogAPI api;

  

    void Start()

    {

        // 创建API实例

        GameObject apiObject = new GameObject("RealtimeDialogAPI");

        api = apiObject.AddComponent<RealtimeDialogClient>();

  

        // 注册事件

        api.OnConnected += () => Debug.Log("已连接");

        api.OnSessionStarted += (dialogId) => Debug.Log($"会话开始: {dialogId}");

        api.OnASRTextReceived += (text, isInterim) => Debug.Log($"ASR: {text}");

        api.OnChatTextReceived += (text, isInterim) => Debug.Log($"Chat: {text}");

  

        // 初始化并连接

        api.Initialize(config);

        api.Connect();

    }

  

    // 当连接成功后会自动触发

    void OnConnected()

    {

        // 开始会话

        api.StartSession();

    }

  

    // 当会话开始后

    void OnSessionStarted(string dialogId)

    {

        // 启动麦克风

        api.StartMicrophone();

  

        // 或发送文本

        // api.SendTextQuery("你好");

    }

  

    void OnDestroy()

    {

        api?.Disconnect();

    }

}

```

  

---

  

## API参考

  

### 核心接口：IRealtimeDialogAPI

  

#### 属性

  

| 属性 | 类型 | 说明 |

|------|------|------|

| `CurrentState` | `RTDState` | 当前状态 |

| `IsConnected` | `bool` | 是否已连接 |

| `IsSessionActive` | `bool` | 会话是否活跃 |

| `SessionId` | `string` | 当前会话ID |

| `IsMicrophoneRecording` | `bool` | 麦克风是否录音中 |

| `IsPlaybackMuted` | `bool` | 播放是否静音 |

  

#### 核心方法

  

##### `void Initialize(RTDConfig config)`

初始化API，必须首先调用。

  

```csharp

api.Initialize(config);

```

  

##### `void Connect()`

连接到服务器。

  

```csharp

api.Connect();

```

  

##### `void Disconnect()`

断开连接。

  

```csharp

api.Disconnect();

```

  

##### `void StartSession()`

开始会话（必须在连接成功后调用）。

  

```csharp

api.StartSession();

```

  

##### `void EndSession()`

结束会话。

  

```csharp

api.EndSession();

```

  

##### `void SendAudioData(byte[] audioData)`

发送音频数据（PCM格式）。

  

```csharp

byte[] pcmData = GetMicrophoneData();

api.SendAudioData(pcmData);

```

  

##### `void SendTextQuery(string text)`

发送文本查询。

  

```csharp

api.SendTextQuery("今天天气怎么样？");

```

  

##### `void SendGreeting(string greeting)`

发送问候消息。

  

```csharp

api.SendGreeting("你好，我是豆包");

```

  

#### 音频控制方法

  

##### `void StartMicrophone()`

启动麦克风录音。

  

```csharp

api.StartMicrophone();

```

  

##### `void StopMicrophone()`

停止麦克风录音。

  

```csharp

api.StopMicrophone();

```

  

##### `void MutePlayback(bool mute)`

静音/取消静音播放。

  

```csharp

api.MutePlayback(true); // 静音

api.MutePlayback(false); // 取消静音

```

  

#### 高级方法

  

##### `void SendAudioFile(string filePath, int chunkSize, int intervalMs)`

发送音频文件（模拟流式发送）。

  

```csharp

api.SendAudioFile("path/to/audio.pcm", 640, 20);

```

  

##### `void SaveReceivedAudio(string filePath)`

保存接收到的音频数据。

  

```csharp

api.SaveReceivedAudio("output.pcm");

```

  

##### `byte[] GetReceivedAudioData()`

获取接收到的音频数据。

  

```csharp

byte[] audioData = api.GetReceivedAudioData();

```

  

#### 事件

  

| 事件 | 签名 | 说明 |

|------|------|------|

| `OnStateChanged` | `event RTDStateCallback` | 状态变化 |

| `OnConnected` | `event Action` | 连接成功 |

| `OnDisconnected` | `event Action` | 连接断开 |

| `OnSessionStarted` | `event Action<string>` | 会话开始（dialog_id） |

| `OnSessionEnded` | `event Action` | 会话结束 |

| `OnAudioReceived` | `event RTDAudioCallback` | 接收到音频 |

| `OnASRTextReceived` | `event RTDTextCallback` | 接收到ASR文本 |

| `OnChatTextReceived` | `event RTDTextCallback` | 接收到聊天文本 |

| `OnUserSpeakingStart` | `event Action<string>` | 用户开始说话 |

| `OnUserSpeakingEnd` | `event Action` | 用户停止说话 |

| `OnTTSStart` | `event Action<string, string>` | TTS开始 |

| `OnTTSEnd` | `event Action` | TTS结束 |

| `OnError` | `event RTDErrorCallback` | 错误事件 |

  

#### 事件使用示例

  

```csharp

// 注册事件

api.OnASRTextReceived += (text, isInterim) => {

    if (!isInterim) {

        Debug.Log($"最终识别结果: {text}");

    } else {

        Debug.Log($"临时识别结果: {text}");

    }

};

  

api.OnChatTextReceived += (text, isInterim) => {

    // 流式接收聊天文本

    chatTextUI.text += text;

};

  

api.OnUserSpeakingStart += (questionId) => {

    Debug.Log("用户开始说话，准备打断TTS");

    api.MutePlayback(true);

};

  

api.OnError += (error, errorCode) => {

    Debug.LogError($"错误[{errorCode}]: {error}");

};

```

  

---

  

## 高级用法

  

### 1. 完整的对话流程

  

```csharp

public class AdvancedDialog : MonoBehaviour

{

    private IRealtimeDialogAPI api;

    private bool autoReconnect = true;

  

    void Start()

    {

        SetupAPI();

    }

  

    void SetupAPI()

    {

        // 创建API

        api = gameObject.AddComponent<RealtimeDialogClient>();

  

        // 注册所有事件

        RegisterAllEvents();

  

        // 初始化并连接

        api.Initialize(config);

        api.Connect();

    }

  

    void RegisterAllEvents()

    {

        // 连接事件

        api.OnConnected += () => {

            Debug.Log("✅ 连接成功");

            api.StartSession();

        };

  

        api.OnDisconnected += () => {

            Debug.Log("❌ 连接断开");

            if (autoReconnect) {

                StartCoroutine(ReconnectAfterDelay(5f));

            }

        };

  

        // 会话事件

        api.OnSessionStarted += (dialogId) => {

            Debug.Log($"✅ 会话开始: {dialogId}");

            api.SendGreeting("你好，我是豆包");

            api.StartMicrophone();

        };

  

        api.OnSessionEnded += () => {

            Debug.Log("⏹️ 会话结束");

            api.StopMicrophone();

        };

  

        // 用户说话事件

        api.OnUserSpeakingStart += (questionId) => {

            Debug.Log("🗣️ 用户开始说话");

            // 打断当前播放

            api.MutePlayback(true);

        };

  

        api.OnUserSpeakingEnd += () => {

            Debug.Log("🤐 用户停止说话");

            api.MutePlayback(false);

        };

  

        // 文本事件

        api.OnASRTextReceived += (text, isInterim) => {

            if (!isInterim) {

                OnASRFinalText(text);

            } else {

                OnASRInterimText(text);

            }

        };

  

        api.OnChatTextReceived += (text, isInterim) => {

            OnChatText(text, isInterim);

        };

  

        // TTS事件

        api.OnTTSStart += (ttsType, text) => {

            Debug.Log($"🔊 TTS开始 [{ttsType}]: {text}");

        };

  

        api.OnTTSEnd += () => {

            Debug.Log("🔇 TTS结束");

        };

  

        // 音频事件

        api.OnAudioReceived += (audioData) => {

            OnAudioReceived(audioData);

        };

  

        // 错误事件

        api.OnError += (error, errorCode) => {

            HandleError(error, errorCode);

        };

    }

  

    IEnumerator ReconnectAfterDelay(float delay)

    {

        yield return new WaitForSeconds(delay);

        if (!api.IsConnected)

        {

            Debug.Log("尝试重新连接...");

            api.Connect();

        }

    }

  

    void OnASRFinalText(string text)

    {

        Debug.Log($"🎤 [最终] {text}");

        // 更新UI显示用户说的话

        userTextUI.text = text;

    }

  

    void OnASRInterimText(string text)

    {

        Debug.Log($"🎤 [临时] {text}");

        // 实时更新UI

        userTextUI.text = text;

    }

  

    void OnChatText(string text, bool isInterim)

    {

        // 流式显示AI回复

        if (isInterim)

        {

            chatTextUI.text += text;

        }

        else

        {

            // 完整文本

            Debug.Log($"💬 [完整] {text}");

        }

    }

  

    void OnAudioReceived(byte[] audioData)

    {

        // 可以保存或处理音频数据

        // 例如：实时可视化音频波形

    }

  

    void HandleError(string error, int errorCode)

    {

        Debug.LogError($"❌ 错误 [{errorCode}]: {error}");

  

        switch (errorCode)

        {

            case 45000002:

                Debug.LogError("错误：上传了空音频");

                break;

            case 45000003:

                Debug.LogError("错误：10分钟静音，连接已释放");

                api.Disconnect();

                break;

            case 55000001:

                Debug.LogError("错误：服务器处理错误");

                break;

            default:

                Debug.LogError($"未知错误代码: {errorCode}");

                break;

        }

    }

  

    void OnDestroy()

    {

        api?.StopMicrophone();

        api?.Disconnect();

    }

}

```

  

### 2. 多音色切换

  

```csharp

public class MultiSpeakerExample : MonoBehaviour

{

    private IRealtimeDialogAPI api;

  

    // 可用音色列表

    private string[] speakers = new string[]

    {

        "zh_female_vv_jupiter_bigtts",      // vv - 活泼灵动女声

        "zh_female_xiaohe_jupiter_bigtts",   // xiaohe - 甜美活泼女声

        "zh_male_yunzhou_jupiter_bigtts",    // yunzhou - 清爽沉稳男声

        "zh_male_xiaotian_jupiter_bigtts"    // xiaotian - 清爽磁性男声

    };

  

    private int currentSpeakerIndex = 0;

  

    public void SwitchToNextSpeaker()

    {

        // 切换音色需要重新开始会话

        api.EndSession();

  

        currentSpeakerIndex = (currentSpeakerIndex + 1) % speakers.Length;

        config.defaultSpeaker = speakers[currentSpeakerIndex];

  

        api.UpdateConfig(config);

        api.StartSession();

  

        Debug.Log($"切换到音色: {speakers[currentSpeakerIndex]}");

    }

}

```

  

### 3. 音频文件处理

  

```csharp

public class AudioFileExample : MonoBehaviour

{

    private IRealtimeDialogAPI api;

  

    public void ProcessAudioFile(string filePath)

    {

        StartCoroutine(ProcessAudioFileRoutine(filePath));

    }

  

    IEnumerator ProcessAudioFileRoutine(string filePath)

    {

        // 确保连接

        if (!api.IsConnected)

        {

            api.Connect();

            yield return new WaitUntil(() => api.IsConnected);

        }

  

        // 确保会话活跃

        if (!api.IsSessionActive)

        {

            api.StartSession();

            yield return new WaitUntil(() => api.IsSessionActive);

        }

  

        // 发送音频文件

        Debug.Log($"开始发送音频文件: {filePath}");

        api.SendAudioFile(filePath, 640, 20);

    }

  

    public void SaveReceivedAudio()

    {

        string timestamp = System.DateTime.Now.ToString("yyyyMMdd_HHmmss");

        string filePath = $"{Application.dataPath}/received_audio_{timestamp}.pcm";

        api.SaveReceivedAudio(filePath);

        Debug.Log($"音频已保存: {filePath}");

    }

}

```

  

### 4. 文本模式使用

  

```csharp

public class TextModeExample : MonoBehaviour

{

    private IRealtimeDialogAPI api;

  

    void Start()

    {

        // 配置为文本模式

        config.inputMode = InputMode.Text;

  

        api = gameObject.AddComponent<RealtimeDialogClient>();

        api.Initialize(config);

  

        // 注册事件

        api.OnSessionStarted += (dialogId) => {

            Debug.Log("会话开始，可以发送文本了");

        };

  

        api.OnChatTextReceived += (text, isInterim) => {

            Debug.Log($"AI回复: {text}");

        };

  

        // 连接并开始会话

        api.Connect();

    }

  

    public void SendUserMessage(string message)

    {

        if (api.IsSessionActive)

        {

            api.SendTextQuery(message);

        }

    }

}

```

  

---

  

## 常见问题

  

### Q1: 如何集成WebSocket库？

  

**A:** 推荐使用NativeWebSocket：

  

1. 在Unity Package Manager中添加：

```

https://github.com/endel/NativeWebSocket.git#upm

```

  

2. 在`RealtimeDialogClient.cs`中实现WebSocket连接：

```csharp

using NativeWebSocket;

  

private WebSocket _webSocket;

  

private async UniTask ConnectWebSocketAsync()

{

    _webSocket = new WebSocket(_config.wsUrl);

  

    _webSocket.OnOpen += () => {

        Debug.Log("WebSocket已打开");

    };

  

    _webSocket.OnMessage += (bytes) => {

        HandleWebSocketMessage(bytes);

    };

  

    _webSocket.OnError += (error) => {

        Debug.LogError($"WebSocket错误: {error}");

    };

  

    _webSocket.OnClose += (code) => {

        Debug.Log($"WebSocket已关闭: {code}");

    };

  

    await _webSocket.Connect();

}

```

  

### Q2: 麦克风权限如何处理？

  

**A:** 在不同平台上处理：

  

**Android:**

在`AndroidManifest.xml`中添加：

```xml

<uses-permission android:name="android.permission.RECORD_AUDIO"/>

<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS"/>

```

  

在代码中请求权限：

```csharp

#if UNITY_ANDROID

if (!Permission.HasUserAuthorizedPermission(Permission.Microphone))

{

    Permission.RequestUserPermission(Permission.Microphone);

}

#endif

```

  

**iOS:**

在`Info.plist`中添加：

```xml

<key>NSMicrophoneUsageDescription</key>

<string>需要访问麦克风以进行语音对话</string>

```

  

### Q3: 如何处理网络断线重连？

  

**A:** 实现自动重连逻辑：

  

```csharp

private int reconnectAttempts = 0;

private int maxReconnectAttempts = 5;

private float reconnectDelay = 5f;

  

api.OnDisconnected += () => {

    if (reconnectAttempts < maxReconnectAttempts)

    {

        reconnectAttempts++;

        Debug.Log($"尝试重新连接 ({reconnectAttempts}/{maxReconnectAttempts})...");

        StartCoroutine(ReconnectAfterDelay(reconnectDelay));

    }

    else

    {

        Debug.LogError("重连失败，已达到最大尝试次数");

    }

};

  

api.OnConnected += () => {

    reconnectAttempts = 0; // 重置计数

};

  

IEnumerator ReconnectAfterDelay(float delay)

{

    yield return new WaitForSeconds(delay);

    api.Connect();

}

```

  

### Q4: 音频格式如何选择？

  

**A:**

- **Pcm_S16LE** (16位小端序PCM)：推荐用于移动设备，节省带宽

- **Pcm_F32LE** (32位浮点PCM)：推荐用于桌面平台，音质更好

  

配置示例：

```csharp

// 移动设备

config.pcmFormat = PcmFormat.Pcm_S16LE;

config.outputSampleRate = 24000;

  

// 桌面设备

config.pcmFormat = PcmFormat.Pcm_F32LE;

config.outputSampleRate = 24000;

```

  

### Q5: 如何实现打断功能？

  

**A:** 监听用户说话事件并静音播放：

  

```csharp

api.OnUserSpeakingStart += (questionId) => {

    // 用户开始说话时打断AI播放

    api.MutePlayback(true);

    Debug.Log("用户说话，AI被打断");

};

  

api.OnTTSEnd += () => {

    // TTS结束后恢复播放

    api.MutePlayback(false);

};

```

  

---

  

## 错误处理

  

### 错误码说明

  

| 错误码 | 说明 | 处理建议 |

|--------|------|----------|

| 45000002 | 空音频包 | 检查麦克风是否正常工作 |

| 45000003 | 10分钟静音超时 | 重新连接 |

| 55000001 | 服务器处理错误 | 检查网络连接，重试 |

| 55000030 | 服务不可用 | 稍后重试 |

| 55002070 | 下游服务错误 | 联系技术支持 |

  

### 错误处理示例

  

```csharp

api.OnError += (error, errorCode) => {

    switch (errorCode)

    {

        case 45000002:

            Debug.LogWarning("检测到空音频，请检查麦克风");

            RestartMicrophone();

            break;

  

        case 45000003:

            Debug.LogWarning("静音超时，重新连接");

            api.Disconnect();

            api.Connect();

            break;

  

        case 55000001:

            Debug.LogError("服务器错误，尝试重连");

            RetryConnection();

            break;

  

        case 55000030:

            Debug.LogError("服务不可用");

            ShowErrorToUser("服务暂时不可用，请稍后重试");

            break;

  

        default:

            Debug.LogError($"未知错误 [{errorCode}]: {error}");

            break;

    }

};

  

void RestartMicrophone()

{

    api.StopMicrophone();

    StartCoroutine(DelayedStartMicrophone(1f));

}

  

IEnumerator DelayedStartMicrophone(float delay)

{

    yield return new WaitForSeconds(delay);

    api.StartMicrophone();

}

```

  

---

  

## 最佳实践

  

### 1. 资源管理

  

```csharp

public class ProperResourceManagement : MonoBehaviour

{

    private IRealtimeDialogAPI api;

  

    void OnEnable()

    {

        InitializeAPI();

    }

  

    void OnDisable()

    {

        CleanupAPI();

    }

  

    void OnApplicationPause(bool pauseStatus)

    {

        if (pauseStatus)

        {

            // 应用暂停时停止麦克风

            api?.StopMicrophone();

        }

        else

        {

            // 应用恢复时重启麦克风

            if (api?.IsSessionActive == true)

            {

                api?.StartMicrophone();

            }

        }

    }

  

    void OnApplicationFocus(bool hasFocus)

    {

        if (!hasFocus)

        {

            // 失去焦点时静音

            api?.MutePlayback(true);

        }

        else

        {

            api?.MutePlayback(false);

        }

    }

  

    void CleanupAPI()

    {

        api?.StopMicrophone();

        api?.EndSession();

        api?.Disconnect();

    }

  

    void OnDestroy()

    {

        CleanupAPI();

    }

}

```

  

### 2. 性能优化

  

```csharp

public class PerformanceOptimization : MonoBehaviour

{

    private IRealtimeDialogAPI api;

  

    void Awake()

    {

        // 限制帧率以降低CPU使用

        Application.targetFrameRate = 60;

  

        // 禁用不必要的日志

        config.enableVerboseLogging = false;

    }

  

    void Start()

    {

        // 使用对象池管理音频缓冲区

        InitializeAudioBufferPool();

    }

  

    void InitializeAudioBufferPool()

    {

        // 实现音频缓冲区对象池

        // 避免频繁分配和释放内存

    }

}

```

  

### 3. UI响应性

  

```csharp

public class ResponsiveUI : MonoBehaviour

{

    [SerializeField] private Text statusText;

    [SerializeField] private Image connectionIndicator;

    [SerializeField] private GameObject loadingSpinner;

  

    void Start()

    {

        api.OnStateChanged += UpdateUIForState;

    }

  

    void UpdateUIForState(RTDState state)

    {

        switch (state)

        {

            case RTDState.Disconnected:

                statusText.text = "未连接";

                connectionIndicator.color = Color.red;

                loadingSpinner.SetActive(false);

                break;

  

            case RTDState.Connecting:

                statusText.text = "连接中...";

                connectionIndicator.color = Color.yellow;

                loadingSpinner.SetActive(true);

                break;

  

            case RTDState.Connected:

                statusText.text = "已连接";

                connectionIndicator.color = Color.green;

                loadingSpinner.SetActive(false);

                break;

  

            case RTDState.SessionActive:

                statusText.text = "会话活跃";

                connectionIndicator.color = Color.green;

                break;

  

            case RTDState.Error:

                statusText.text = "错误";

                connectionIndicator.color = Color.red;

                loadingSpinner.SetActive(false);

                break;

        }

    }

}

```

  

### 4. 日志管理

  

```csharp

public class LoggingBestPractices : MonoBehaviour

{

    private IRealtimeDialogAPI api;

  

    void Start()

    {

        // 开发环境启用详细日志

        #if UNITY_EDITOR || DEVELOPMENT_BUILD

        config.enableVerboseLogging = true;

        #else

        config.enableVerboseLogging = false;

        #endif

  

        // 记录关键事件

        api.OnConnected += () => LogEvent("Connected");

        api.OnSessionStarted += (dialogId) => LogEvent($"SessionStarted: {dialogId}");

        api.OnError += (error, code) => LogError($"Error[{code}]: {error}");

    }

  

    void LogEvent(string message)

    {

        string timestamp = System.DateTime.Now.ToString("HH:mm:ss.fff");

        Debug.Log($"[{timestamp}] {message}");

    }

  

    void LogError(string message)

    {

        string timestamp = System.DateTime.Now.ToString("HH:mm:ss.fff");

        Debug.LogError($"[{timestamp}] {message}");

  

        // 可以将错误发送到分析服务

        // AnalyticsService.LogError(message);

    }

}

```

  

---

  

## 附录

  

### A. 配置参数完整说明

  

| 参数 | 类型 | 默认值 | 说明 |

|------|------|--------|------|

| `wsUrl` | string | wss://openspeech.bytedance.com/api/v3/realtime/dialogue | WebSocket服务器地址 |

| `apiAppId` | string | - | 火山引擎APP ID（必填） |

| `apiAccessKey` | string | - | 火山引擎Access Key（必填） |

| `apiAppKey` | string | PlgvMymc7f3tQnJ6 | API应用Key（固定值） |

| `inputSampleRate` | int | 16000 | 输入采样率（Hz） |

| `outputSampleRate` | int | 24000 | 输出采样率（Hz） |

| `channels` | int | 1 | 声道数 |

| `pcmFormat` | PcmFormat | Pcm_S16LE | PCM格式 |

| `defaultSpeaker` | string | zh_female_vv_jupiter_bigtts | 默认发音人 |

| `audioChunkSize` | int | 640 | 音频块大小（字节） |

| `audioSendInterval` | int | 20 | 发送间隔（毫秒） |

| `inputMode` | InputMode | Audio | 输入模式 |

| `modelVersion` | ModelVersion | O | 模型版本 |

| `botName` | string | 豆包 | 机器人名称 |

| `systemRole` | string | - | 系统角色描述 |

| `speakingStyle` | string | - | 说话风格 |

| `enableVerboseLogging` | bool | true | 启用详细日志 |

| `saveReceivedAudio` | bool | false | 保存接收的音频 |

  

### B. 事件ID参考

  

#### 客户端事件

- `1` - StartConnection

- `2` - FinishConnection

- `100` - StartSession

- `102` - FinishSession

- `200` - TaskRequest (音频上传)

- `300` - SayHello

- `500` - ChatTTSText

- `501` - ChatTextQuery

- `502` - ChatRAGText

  

#### 服务端事件

- `50` - ConnectionStarted

- `51` - ConnectionFailed

- `52` - ConnectionFinished

- `150` - SessionStarted

- `152` - SessionFinished

- `153` - SessionFailed

- `154` - UsageResponse

- `350` - TTSSentenceStart

- `351` - TTSSentenceEnd

- `352` - TTSResponse (音频数据)

- `359` - TTSEnded

- `450` - ASRInfo

- `451` - ASRResponse

- `459` - ASREnded

- `550` - ChatResponse

- `553` - ChatTextQueryConfirmed

- `559` - ChatEnded

- `599` - DialogCommonError

  

### C. 音色列表

  

#### O版本音色

- `zh_female_vv_jupiter_bigtts` - vv（活泼灵动女声）

- `zh_female_xiaohe_jupiter_bigtts` - xiaohe（甜美活泼女声）

- `zh_male_yunzhou_jupiter_bigtts` - yunzhou（清爽沉稳男声）

- `zh_male_xiaotian_jupiter_bigtts` - xiaotian（清爽磁性男声）

  

#### SC版本音色（克隆音色）

请参考火山引擎官方文档获取完整的克隆音色列表。

  

---

  

## 技术支持

  

如有问题，请联系：

- 火山引擎官方文档：https://www.volcengine.com/docs/

- GitHub Issues：(项目地址)

- 邮箱支持：(联系邮箱)

  

---

  

**最后更新：2025年1月**

**版本：1.0.0**