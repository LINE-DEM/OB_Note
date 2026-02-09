# SileroVADManager 错误修复文档

## 文档信息
- **日期**: 2026-01-16
- **项目**: YiTiJi_Move
- **文件**: Assets/c#/System/VAD/SileroVADManager.cs
- **模型**: Silero VAD v4/v6 (ONNX)

---

## 问题描述

### 症状
- VAD 系统正常运行,处理帧速度正常 (~0.3ms/帧)
- 麦克风成功启动并捕获音频数据
- **但无论说话与否,语音概率始终接近 0 (0.0005-0.0012)**
- 从未触发语音检测事件

### 日志表现
```
[VAD] Audio stats | Max: 0.106171 | Avg: 0.025879 | Samples: 640  ✅ 音频正常
[VAD] Processed 300 frames | Raw probability: 0.0005  ❌ 概率异常
[VAD] Smoothed probability: 0.0005 | Threshold: 0.50 | Active: False
```

### 关键异常指标
LSTM 状态值异常增长:
```
State update | Old max: 99.000000 | New max: 100.000000
State update | Old max: 199.000000 | New max: 200.000000
State update | Old max: 299.000000 | New max: 300.000000
```
**正常的 LSTM 状态应该在 -1 到 1 之间!**

---

## 根本原因

### 问题定位
通过分析 Silero VAD v6.2 官方源码 (`utils_vad.py`),发现了关键问题:

**模型期望的输入格式与代码实际传入的格式不匹配!**

### 官方源码关键代码 (utils_vad.py)
```python
# 第 66 行: 计算 context 大小
context_size = 64 if sr == 16000 else 32

# 第 76 行: 初始化 context
self._context = torch.zeros(batch_size, context_size)

# 第 78 行: 【关键】拼接 context + audio
x = torch.cat([self._context, x], dim=1)

# 第 87 行: 更新 context
self._context = x[..., -context_size:]
```

### 输入格式对比

| 参数       | 官方要求                     | 原代码          | 问题           |
| -------- | ------------------------ | ------------ | ------------ |
| input 形状 | **[1, 576]**             | [1, 512]     | ❌ 缺少 64 个样本  |
| input 内容 | context(64) + audio(512) | 仅 audio(512) | ❌ 缺少 context |
| state 形状 | [2, 1, 128]              | [2, 1, 128]  | ✅ 正确         |
| sr 格式    | int64                    | int64        | ✅ 正确         |
|          |                          |              |              |

### 根本原因总结
**原代码只传入了 512 个音频样本,但模型期望接收 576 个样本 (64 个 context + 512 个音频)**

缺少 context 导致:
1. 模型内部计算错误
2. LSTM 状态值异常增长
3. 输出概率始终接近 0

---

## 修复方案

### 修复思路
1. 添加 context 缓冲区管理
2. 每帧输入前拼接 context + audio
3. 每帧推理后更新 context
4. 重置状态时同时重置 context

### 代码修改

#### 1. 添加常量和字段 (第 50-82 行)
```csharp
// Context 常量
private const int CONTEXT_SIZE_16K = 64;  // 16kHz 采样率
private const int CONTEXT_SIZE_8K = 32;   // 8kHz 采样率

// Context 缓冲区
private float[] context;
private int contextSize;
```

#### 2. 初始化 Context (InitializeVAD 方法, 第 215-221 行)
```csharp
// 初始化 Context 缓冲区
contextSize = (sampleRate == 16000) ? CONTEXT_SIZE_16K : CONTEXT_SIZE_8K;
context = new float[contextSize];
Debug.Log($"Context initialized with size: {contextSize}");
```

#### 3. 构建完整输入 (ProcessFrame 方法, 第 536-563 行)
```csharp
// 构建完整输入: context + audio = 576 样本
int fullInputSize = contextSize + frameSize;  // 64 + 512 = 576
float[] fullInput = new float[fullInputSize];

// 拼接
Array.Copy(context, 0, fullInput, 0, contextSize);
Array.Copy(normalizedAudio, 0, fullInput, contextSize, frameSize);

// 创建张量 - 注意形状是 [1, 576]
var inputTensor = new DenseTensor<float>(
    fullInput,
    new int[] { 1, fullInputSize }
);
```

#### 4. 更新 Context (ProcessFrame 方法, 第 627-632 行)
```csharp
// 保存最后 64 个样本作为下一帧的 context
Array.Copy(fullInput, fullInput.Length - contextSize, context, 0, contextSize);
```

#### 5. 重置 Context (ResetState 方法, 第 674-693 行)
```csharp
public void ResetState()
{
    if (state != null) Array.Clear(state, 0, state.Length);
    if (context != null) Array.Clear(context, 0, context.Length);  // 新增
    recentProbabilities?.Clear();
    isVoiceActive = false;
    wasVoiceActive = false;
}
```

---

## 修复结果

### 修复前
- 概率: 0.0005 (恒定)
- 状态: 异常增长 (99, 199, 299...)
- 检测: 永不触发

### 修复后
- 概率: 说话时 > 0.5,沉默时 < 0.3
- 状态: 正常范围 (-1 到 1)
- 检测: 正常触发语音开始/结束事件

---

## 经验教训

### 1. 仔细阅读官方源码
Silero VAD 的官方 Python 实现包含了所有关键细节。在移植到其他语言时,必须完全理解原始实现。

### 2. 关注异常指标
LSTM 状态值异常增长 (99, 199, 299...) 是关键线索,表明输入格式有问题。

### 3. Context 机制是 Silero VAD 的核心特性
- 16kHz 采样率: 64 个 context 样本
- 8kHz 采样率: 32 个 context 样本
- 必须在每帧音频前添加 context
- 必须在每帧推理后更新 context

### 4. 张量形状检查
移植神经网络模型时,始终验证:
- 输入张量形状
- 输入张量内容
- 输出张量形状

---

## 相关文件

| 文件 | 说明 |
|------|------|
| SileroVADManager.cs | 主要 VAD 实现文件 |
| silero_vad.onnx | Silero VAD ONNX 模型 |
| utils_vad.py | Silero VAD 官方 Python 实现 (参考) |

---

## 参考资料

- Silero VAD GitHub: https://github.com/snakers4/silero-vad
- 官方源码路径: C:\1.ATemp\silero-vad-6.2\silero-vad-6.2\src\silero_vad\utils_vad.py
- 关键行号: 51-56 (reset_states), 66-80 (__call__), 87 (context update)

---

**文档结束**
