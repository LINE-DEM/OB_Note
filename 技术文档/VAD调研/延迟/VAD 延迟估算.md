# VAD 延迟估算（从开口到判定“有人声”）

  

本文只讨论 **VAD 检测“有人声/静音”的延迟**（即 `SileroVADManager.OnVoiceActivityChanged(true/false)` 触发时刻）。  

不讨论“ASR 出文字”的端到端延迟（那会额外叠加 WebSocket 建连、说话时长、尾静音确认、服务端结算等）。

  

---

  

## 结论（给业务/测试的可用数字）

  

在当前实现（`Assets\\c#\\System\\VAD\\SileroVADManager.cs`）默认配置下：

  

- **(1) VAD 模型推理时间（单帧）**：通常是 **亚毫秒级（< 1ms）**，代码里用 `Stopwatch` 统计到 `lastProcessingTime`。  

  - 推理粒度：每次推理输入 **512 samples**（16kHz 下约 **32ms 音频**）。

- **(2) VAD 管理器确认“有人声”所需的音频缓冲/历史**：通常需要 **约 1–2 帧**的稳定人声（约 **32–64ms**），再叠加音频采集回调/线程调度等开销。  

  - 实际从“开口”到 `OnVoiceActivityChanged(true)` **更常见的量级是 ~150–200ms**（主要由 NAudio 采集回调的 chunk 周期决定）。

  

> 如果你看到 VAD 反应“慢一拍”，大概率不是模型推理慢，而是 **采集回调的缓冲周期 + 平滑窗口 + 线程切主线程** 叠加导致。

  

---

  

## 1) VAD 大模型的时间：从输入音频到有结果

  

### 1.1 推理粒度（输入音频有多长）

  

配置在 `SileroVADManager`：

  

- `sampleRate = 16000`

- `frameSize = 512`（tooltip 也写了“Silero 推荐 512”）

  

因此每帧音频时长：

  

```

frameDuration = frameSize / sampleRate

              = 512 / 16000

              = 0.032s ≈ 32ms

```

  

### 1.2 代码里实际统计的“模型耗时”

  

`SileroVADManager.ProcessingLoop()` 对每帧执行：

  

- `float probability = ProcessFrame(frame);`

- 用 `Stopwatch` 计时并赋值到 `lastProcessingTime`（单位 ms）

  

所以 “VAD 大模型时间” 对应的是：

  

> **每 32ms 音频** → 产出一个 `probability`（0~1）所花的 CPU 时间（`lastProcessingTime`）。

  

在 PC/常规 CPU 上，Silero VAD 的 ONNX 推理通常是 **亚毫秒级**；如果你想给出你们设备的真实数值，建议直接在运行日志里抓 `lastProcessingTime`（工程里每 100 帧会打印一次统计日志）。

  

---

  

## 2) VAD 管理器缓冲多久，才确认是人的声音

  

这里分三层“缓冲/等待”：

  

1) **音频采集回调的 chunk 周期（最关键）**  

2) **VAD 自身帧切分 + 平滑窗口（决定需要多少历史）**  

3) **线程/主线程调度（Unity 帧率相关）**

  

### 2.1 音频采集（NAudio）的 chunk 周期

  

`SileroVADManager.StartMicrophone()` 使用 `NAudio.Wave.WaveInEvent`：

  

- 只设置了 `waveIn.WaveFormat = 16kHz/16bit/mono`

- **没有设置** `waveIn.BufferMilliseconds`

  

NAudio 的 `WaveInEvent` 默认 `BufferMilliseconds` 通常是 **100ms**，并且本文件里也有明显暗示：

  

- `rawAudioBuffer = new byte[20 * 3200]`

- 16kHz * 16bit * mono = 32000 bytes/s  

  - 100ms ≈ 3200 bytes（与代码常量吻合）

  

因此，**从你开始说话到程序“第一次拿到可处理的音频块”**，常见下限就是 **~100ms**（取决于系统音频驱动与 NAudio 回调节奏）。

  

### 2.2 帧切分（32ms）+ 平滑窗口（2帧）

  

在 `SileroVADManager.Update()` 中：

  

- 当 `rawAudioBuffer.Length >= frameSize*2`（即 ≥1024 bytes）才会开始把音频拆成帧并入队

- 每帧 512 samples ≈ 32ms

  

之后在 `ProcessingLoop()`：

  

- 对每帧得到 `probability`

- 经过 `ApplySmoothing(probability)` 做移动平均平滑

  

平滑窗口参数：

  

- `smoothWindowSize = 2`（tooltip 明确写了“2帧=64ms延迟”）

  

这意味着 VAD 的“是否有人声”不是只看**当前帧**，而是看**最近两帧的平均值**。  

在“从静音进入说话”的边界处，第一帧往往会被前一帧静音概率拉低，导致需要 **约 2 帧（≈64ms）** 才稳定越过阈值。

  

所以就“VAD 自己需要多少音频历史才能确认人声”而言，可以粗略记成：

  

- **至少 1 帧（32ms）**才能出一次判定  

- **更常见需要 ~2 帧（64ms）** 才会从 `false→true`（因为有平滑 + 阈值）

  

### 2.3 线程/调度带来的额外等待

  

在当前实现里：

  

- 入队在 `SileroVADManager.Update()`（主线程）

- 推理在 `ProcessingLoop()`（后台线程）

- 当队列为空时，后台线程会 `Thread.Sleep(10)` 避免空转  

  - 因此“新音频刚入队”到“处理线程真正开始处理”额外延迟理论上 **0~10ms**

- `OnVoiceActivityChanged` 事件在后台线程触发；业务层（如 `VADASRController`）通常还会 `MainThreadDispatcher.Enqueue(...)` 切回主线程  

  - 这会再增加 **0~1 帧**的等待（例如 60fps 下约 **0~16ms**）

  

---

  

## 3) 把这些拼起来：从“开口”到 VAD 报告“有人声”的总延迟

  

用“典型值”粗估：

  

```



```

  

你也可以把它理解成：

  

- **真正的瓶颈通常是采集 chunk（100ms）**，不是模型推理。

- 如果要更“跟手”，最有效的是缩短采集 chunk（例如显式调低 `WaveInEvent.BufferMilliseconds`），其次才是调整平滑窗口与阈值策略。

  

---

  

## 4)（可选）如果要进一步精确：建议怎么测

  

如果你需要可复现的数值（而不是估算）：

  

1) 在 `OnNAudioDataAvailable` 里记录回调时间戳与 `BytesRecorded`  

2) 在 `ProcessingLoop` 里记录每帧被处理的时间戳（已经有 `Stopwatch`）  

3) 在订阅侧（如 `VADASRController.HandleVADStateChanged/HandleLongDialogVAD`）记录主线程收到事件的时间戳  

  

这样能把延迟拆成：采集 → 入队 → 推理 → 事件触发 → 主线程收到，定位“慢”的真实来源。





