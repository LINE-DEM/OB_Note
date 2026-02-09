# AudioBufferManager.cs 流程图与说明

  

目标：描述 `AudioBufferManager` 的采集、缓冲、读取与销毁流程，便于理解 VAD 音频链路的上游数据来源。

  

## 1. 主流程（Mermaid）

  

```mermaid

flowchart TD

  A[Awake] --> B[StartRecording]

  B --> C{录音启动成功?}

  C -- 否 --> D[记录错误日志\nisRecording=false]

  C -- 是 --> E[订阅 DataAvailable\nStartRecording\nisRecording=true]

  E --> F[OnAudioDataAvailable]

  F --> G{数据有效?}

  G -- 否 --> H[直接返回]

  G -- 是 --> I[复制 BytesRecorded 到新数组]

  I --> J[AddAudioData]

  J --> K[加锁 bufferLock]

  K --> L[AddRange 追加到音频缓冲]

  L --> M{缓冲长度 > BUFFER_SIZE_BYTES?}

  M -- 是 --> N[移除头部字节\n保持固定大小]

  M -- 否 --> O[保持不变]

  N --> P[释放锁]

  O --> P[释放锁]

```

  

## 2. 读取接口流程

  

```mermaid

flowchart TD

  A[外部调用 ReadFromHead(sampleCount)] --> B[bytesNeeded = sampleCount * 2]

  B --> C[加锁 bufferLock]

  C --> D{缓冲长度 >= bytesNeeded?}

  D -- 否 --> E[返回 null]

  D -- 是 --> F[CopyTo 头部 bytesNeeded]

  F --> G[返回字节数组]

```

  

```mermaid

flowchart TD

  A[外部调用 CopyAllBuffer] --> B[加锁 bufferLock]

  B --> C[ToArray 返回拷贝]

```

  

```mermaid

flowchart TD

  A[外部调用 HasEnoughData(sampleCount)] --> B[bytesNeeded = sampleCount * 2]

  B --> C[加锁 bufferLock]

  C --> D[返回 Count >= bytesNeeded]

```

  

## 3. 退出与清理流程

  

```mermaid

flowchart TD

  A[OnDestroy] --> B{waveIn != null?}

  B -- 否 --> E[跳过]

  B -- 是 --> C[StopRecording + Dispose]

  C --> D[置空 waveIn\n记录日志]

  D --> E[加锁清空 audioBuffer]

  E --> F[结束]

```

  

## 4. 关键点说明

- 音频采集使用 NAudio `WaveInEvent`，采样率 16kHz、16-bit、单声道。

- 缓冲区采用 `List<byte>`，通过 `bufferLock` 保证线程安全。

- `BUFFER_SIZE_BYTES` 固定为 16000（约 0.5 秒音频），超过时从头部丢弃旧数据。

- `ReadFromHead` 仅复制数据，不会消费/移除数据。