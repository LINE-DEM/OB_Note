# ONNX Runtime 大模型接入完整指南

## 目录

1. [核心概念理解](https://claude-d-ai-s-cld.p-tga.aiwentu.com/chat/ba9c62e0-1678-4c37-8f2f-a8579667f90e#%E6%A0%B8%E5%BF%83%E6%A6%82%E5%BF%B5%E7%90%86%E8%A7%A3)
2. [大模型推理工作原理](https://claude-d-ai-s-cld.p-tga.aiwentu.com/chat/ba9c62e0-1678-4c37-8f2f-a8579667f90e#%E5%A4%A7%E6%A8%A1%E5%9E%8B%E6%8E%A8%E7%90%86%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86)
3. [ONNX Runtime 工作流程](https://claude-d-ai-s-cld.p-tga.aiwentu.com/chat/ba9c62e0-1678-4c37-8f2f-a8579667f90e#onnx-runtime-%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B)
4. [参数传入与输出获取](https://claude-d-ai-s-cld.p-tga.aiwentu.com/chat/ba9c62e0-1678-4c37-8f2f-a8579667f90e#%E5%8F%82%E6%95%B0%E4%BC%A0%E5%85%A5%E4%B8%8E%E8%BE%93%E5%87%BA%E8%8E%B7%E5%8F%96)
5. [Unity 完整接入示例](https://claude-d-ai-s-cld.p-tga.aiwentu.com/chat/ba9c62e0-1678-4c37-8f2f-a8579667f90e#unity-%E5%AE%8C%E6%95%B4%E6%8E%A5%E5%85%A5%E7%A4%BA%E4%BE%8B)
6. [工程实践最佳实践](https://claude-d-ai-s-cld.p-tga.aiwentu.com/chat/ba9c62e0-1678-4c37-8f2f-a8579667f90e#%E5%B7%A5%E7%A8%8B%E5%AE%9E%E8%B7%B5%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5)
7. [调试与问题排查](https://claude-d-ai-s-cld.p-tga.aiwentu.com/chat/ba9c62e0-1678-4c37-8f2f-a8579667f90e#%E8%B0%83%E8%AF%95%E4%B8%8E%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5)

---

## 核心概念理解

### 什么是 ONNX?

**ONNX** (Open Neural Network Exchange) 是一个开放的神经网络模型交换格式。想象它是一种"通用语言",让不同框架训练的模型都能被统一理解和运行。

```
训练阶段:
PyTorch 模型 ──┐
TensorFlow 模型 ├──> 导出为 ONNX 格式 (.onnx 文件)
Keras 模型 ────┘

推理阶段:
ONNX 模型 ──> ONNX Runtime ──> 在任何平台运行
```

### 什么是 ONNX Runtime?

**ONNX Runtime** 是一个高性能的推理引擎,负责加载 ONNX 模型并执行推理计算。它就像一个"翻译器 + 执行器":

- **翻译器**: 读懂 ONNX 模型文件中的神经网络结构
- **执行器**: 高效地执行模型中定义的所有数学运算

### 关键术语对照表

|术语|通俗理解|技术定义|
|---|---|---|
|**Session**|模型运行环境|加载模型后创建的推理会话对象|
|**Tensor**|多维数组|模型输入/输出的数据容器|
|**Input**|喂给模型的数据|模型需要的输入数据(如图像、文本)|
|**Output**|模型计算结果|模型产生的输出(如分类结果、概率)|
|**Shape**|数据形状|数组的维度,如 [1, 3, 224, 224]|
|**Node**|计算节点|模型中的一个运算操作|

---

## 大模型推理工作原理

### 整体流程图

```
┌─────────────────────────────────────────────────────────────┐
│                     1. 模型加载阶段                          │
│  .onnx 文件 ──> 读取模型结构 ──> 分配内存 ──> 创建 Session  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     2. 数据准备阶段                          │
│  原始数据 ──> 预处理 ──> 转换为 Tensor ──> 匹配输入格式     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     3. 推理执行阶段                          │
│  输入 Tensor ──> 前向传播计算 ──> 生成输出 Tensor           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     4. 结果处理阶段                          │
│  输出 Tensor ──> 后处理 ──> 转换为可用结果                  │
└─────────────────────────────────────────────────────────────┘
```

### 以语音检测(VAD)模型为例

假设你有一个 Silero VAD 模型用于检测语音:

**模型信息**:

- 输入: 音频片段 (float 数组)
- 输出: 语音概率 (0.0-1.0)

**详细执行过程**:

```
步骤 1: 加载模型
┌──────────────────────────────────────────┐
│ var session = new InferenceSession(      │
│     "silero_vad.onnx"                     │
│ );                                        │
│                                           │
│ 内部发生:                                 │
│ • 解析 .onnx 文件的 protobuf 结构        │
│ • 构建计算图 (有哪些层、连接关系)        │
│ • 为每一层分配内存                        │
│ • 初始化权重参数                          │
└──────────────────────────────────────────┘

步骤 2: 准备输入数据
┌──────────────────────────────────────────┐
│ // 原始音频数据                           │
│ float[] audioSamples = GetAudioChunk();   │
│                                           │
│ // 转换为模型需要的格式                   │
│ var tensor = new DenseTensor<float>(      │
│     audioSamples,                         │
│     new[] { 1, audioSamples.Length }      │
│ );                                        │
│                                           │
│ 内部结构:                                 │
│ Shape: [batch_size, sequence_length]      │
│        [1, 512]  ← 一批数据,512个采样点   │
└──────────────────────────────────────────┘

步骤 3: 执行推理
┌──────────────────────────────────────────┐
│ var inputs = new List<NamedOnnxValue> {   │
│     NamedOnnxValue.CreateFromTensor(      │
│         "input", tensor                   │
│     )                                     │
│ };                                        │
│                                           │
│ var outputs = session.Run(inputs);        │
│                                           │
│ 模型内部计算流程:                         │
│ Input Layer                               │
│    ↓                                      │
│ Conv1D Layer (提取特征)                   │
│    ↓                                      │
│ LSTM Layer (时序分析)                     │
│    ↓                                      │
│ Dense Layer (全连接)                      │
│    ↓                                      │
│ Sigmoid (输出概率)                        │
└──────────────────────────────────────────┘

步骤 4: 获取结果
┌──────────────────────────────────────────┐
│ var output = outputs.First();             │
│ var probability = output                  │
│     .AsEnumerable<float>()                │
│     .First();                             │
│                                           │
│ // probability = 0.85                     │
│ // 表示 85% 的概率检测到语音              │
└──────────────────────────────────────────┘
```

---

## ONNX Runtime 工作流程

### 三层架构理解

```
┌─────────────────────────────────────────────────────┐
│              应用层 (你的代码)                       │
│  • 加载模型                                          │
│  • 准备数据                                          │
│  • 调用推理                                          │
│  • 处理结果                                          │
└─────────────────────────────────────────────────────┘
                        ↕️
┌─────────────────────────────────────────────────────┐
│         C# 托管层 (Microsoft.ML.OnnxRuntime.dll)    │
│  • 提供友好的 C# API                                 │
│  • 封装 P/Invoke 调用                                │
│  • 内存管理和类型转换                                │
└─────────────────────────────────────────────────────┘
                        ↕️ (P/Invoke)
┌─────────────────────────────────────────────────────┐
│         C++ 原生层 (onnxruntime.dll)                 │
│  • 实际的推理引擎                                    │
│  • 图优化                                            │
│  • 算子执行                                          │
│  • 硬件加速 (CPU/GPU)                                │
└─────────────────────────────────────────────────────┘
```

### Session 生命周期管理

```csharp
// ===== 阶段 1: 创建 Session =====
var sessionOptions = new SessionOptions();

// 可选配置
sessionOptions.GraphOptimizationLevel = 
    GraphOptimizationLevel.ORT_ENABLE_ALL;  // 启用所有优化

var session = new InferenceSession(
    "path/to/model.onnx",
    sessionOptions
);

// ===== 阶段 2: 查询模型信息 =====
// 获取输入信息
foreach (var input in session.InputMetadata)
{
    Debug.Log($"输入名称: {input.Key}");
    Debug.Log($"数据类型: {input.Value.ElementType}");
    Debug.Log($"形状: {string.Join(",", input.Value.Dimensions)}");
}

// 获取输出信息
foreach (var output in session.OutputMetadata)
{
    Debug.Log($"输出名称: {output.Key}");
    // ... 类似的信息
}

// ===== 阶段 3: 执行推理 (可多次) =====
for (int i = 0; i < 1000; i++)
{
    var inputs = PrepareInputs();
    var outputs = session.Run(inputs);
    ProcessOutputs(outputs);
    
    // 注意: 每次 Run 后要 Dispose outputs
    foreach (var output in outputs)
    {
        output.Dispose();
    }
}

// ===== 阶段 4: 清理资源 =====
session.Dispose();
sessionOptions.Dispose();
```

---

## 参数传入与输出获取

### 输入参数详解

#### 1️⃣ 理解输入的"形状"(Shape)

```
单个图像输入:
[batch, channels, height, width]
[1,     3,        224,    224   ]
 │      │         │       │
 │      │         │       └─ 图像宽度 224 像素
 │      │         └───────── 图像高度 224 像素
 │      └─────────────────── 3 个颜色通道 (RGB)
 └────────────────────────── 一批数据中有 1 张图像

音频序列输入:
[batch, sequence_length]
[1,     512              ]
 │      │
 │      └─ 512 个音频采样点
 └──────── 一批数据

文本 Token 输入:
[batch, max_token_length]
[1,     128              ]
 │      │
 │      └─ 最多 128 个词
 └──────── 一批句子
```

#### 2️⃣ 创建输入 Tensor

```csharp
// ===== 方法 1: 从一维数组创建 =====
float[] data = new float[512];  // 你的数据
var shape = new int[] { 1, 512 };  // 定义形状

var tensor = new DenseTensor<float>(data, shape);

// ===== 方法 2: 直接指定形状创建空 Tensor =====
var tensor = new DenseTensor<float>(new[] { 1, 3, 224, 224 });

// 手动填充数据
for (int c = 0; c < 3; c++)
{
    for (int h = 0; h < 224; h++)
    {
        for (int w = 0; w < 224; w++)
        {
            tensor[0, c, h, w] = GetPixelValue(c, h, w);
        }
    }
}

// ===== 方法 3: 多维数组直接创建 =====
float[,] matrix = new float[10, 20];  // 2D 数据
var tensor = new DenseTensor<float>(
    matrix.Cast<float>().ToArray(),
    new[] { 10, 20 }
);
```

#### 3️⃣ 构建输入列表

```csharp
// 模型可能有多个输入
var inputs = new List<NamedOnnxValue>();

// 输入 1: 音频数据
var audioTensor = new DenseTensor<float>(audioData, new[] { 1, 512 });
inputs.Add(NamedOnnxValue.CreateFromTensor("input", audioTensor));

// 输入 2: 状态向量 (如果是 RNN/LSTM 模型)
var stateTensor = new DenseTensor<float>(stateData, new[] { 1, 128 });
inputs.Add(NamedOnnxValue.CreateFromTensor("state", stateTensor));

// 输入 3: 序列长度
var lengthTensor = new DenseTensor<long>(new long[] { 512 }, new[] { 1 });
inputs.Add(NamedOnnxValue.CreateFromTensor("seq_len", lengthTensor));
```

### 输出结果详解

#### 1️⃣ 获取输出数据

```csharp
// 执行推理
using (var results = session.Run(inputs))
{
    // ===== 方法 1: 按索引获取 =====
    var firstOutput = results[0];
    
    // ===== 方法 2: 按名称获取 =====
    var namedOutput = results.First(x => x.Name == "output");
    
    // ===== 方法 3: 遍历所有输出 =====
    foreach (var output in results)
    {
        Debug.Log($"输出名称: {output.Name}");
        ProcessOutput(output);
    }
}
```

#### 2️⃣ 提取输出的值

```csharp
// ===== 单个值输出 (分类概率) =====
var output = results.First();
var probability = output.AsEnumerable<float>().First();
Debug.Log($"概率: {probability}");

// ===== 一维数组输出 (如 10 个分类的得分) =====
var scores = output.AsEnumerable<float>().ToArray();
for (int i = 0; i < scores.Length; i++)
{
    Debug.Log($"类别 {i}: {scores[i]}");
}

// ===== 多维数组输出 (如图像分割结果) =====
var tensor = output.AsTensor<float>();
int[] shape = tensor.Dimensions.ToArray();
// shape = [1, 21, 512, 512]  // batch, classes, height, width

// 访问特定位置
float pixelClass = tensor[0, 5, 100, 200];  // 第5类,坐标(100,200)

// ===== 复杂结构输出 (如检测框) =====
// 假设输出是 [N, 6] 其中 6 = [x1, y1, x2, y2, score, class]
var detections = output.AsTensor<float>();
int numDetections = detections.Dimensions[0];

for (int i = 0; i < numDetections; i++)
{
    float x1 = detections[i, 0];
    float y1 = detections[i, 1];
    float x2 = detections[i, 2];
    float y2 = detections[i, 3];
    float score = detections[i, 4];
    float classId = detections[i, 5];
    
    Debug.Log($"检测框 {i}: ({x1},{y1})-({x2},{y2}), " +
              $"置信度={score}, 类别={classId}");
}
```

#### 3️⃣ 类型转换技巧

```csharp
// ===== 处理不同数据类型 =====
var output = results.First();

// 检查类型
if (output.ElementType == TensorElementType.Float)
{
    var floatData = output.AsEnumerable<float>();
}
else if (output.ElementType == TensorElementType.Int64)
{
    var longData = output.AsEnumerable<long>();
}

// ===== Tensor 转换为常用格式 =====
// 转为 Unity Texture2D
Texture2D ConvertToTexture(NamedOnnxValue output, int width, int height)
{
    var tensor = output.AsTensor<float>();
    var texture = new Texture2D(width, height);
    
    for (int y = 0; y < height; y++)
    {
        for (int x = 0; x < width; x++)
        {
            float r = tensor[0, 0, y, x];
            float g = tensor[0, 1, y, x];
            float b = tensor[0, 2, y, x];
            texture.SetPixel(x, y, new Color(r, g, b));
        }
    }
    
    texture.Apply();
    return texture;
}

// 转为 AudioClip 数据
float[] ConvertToAudio(NamedOnnxValue output)
{
    return output.AsEnumerable<float>().ToArray();
}
```

---

## Unity 完整接入示例

### 场景 1: 图像分类模型

```csharp
using UnityEngine;
using Microsoft.ML.OnnxRuntime;
using Microsoft.ML.OnnxRuntime.Tensors;
using System.Collections.Generic;
using System.Linq;

public class ImageClassifier : MonoBehaviour
{
    private InferenceSession session;
    private string[] classLabels;
    
    void Start()
    {
        // 1. 加载模型
        string modelPath = Application.streamingAssetsPath + "/resnet50.onnx";
        session = new InferenceSession(modelPath);
        
        // 2. 加载类别标签
        classLabels = new string[] {
            "猫", "狗", "鸟", "鱼", "马"
            // ... 1000 个 ImageNet 类别
        };
        
        Debug.Log("模型加载成功");
    }
    
    public string ClassifyImage(Texture2D image)
    {
        // ===== 步骤 1: 预处理图像 =====
        var preprocessed = PreprocessImage(image);
        
        // ===== 步骤 2: 创建输入 Tensor =====
        var inputTensor = new DenseTensor<float>(
            preprocessed,
            new[] { 1, 3, 224, 224 }  // [batch, channels, height, width]
        );
        
        var inputs = new List<NamedOnnxValue> {
            NamedOnnxValue.CreateFromTensor("input", inputTensor)
        };
        
        // ===== 步骤 3: 执行推理 =====
        using (var results = session.Run(inputs))
        {
            // ===== 步骤 4: 处理输出 =====
            var output = results.First().AsEnumerable<float>().ToArray();
            
            // Softmax 转换为概率
            var probabilities = Softmax(output);
            
            // 找到最大概率的类别
            int maxIndex = 0;
            float maxProb = 0;
            for (int i = 0; i < probabilities.Length; i++)
            {
                if (probabilities[i] > maxProb)
                {
                    maxProb = probabilities[i];
                    maxIndex = i;
                }
            }
            
            return $"{classLabels[maxIndex]} (置信度: {maxProb:P})";
        }
    }
    
    private float[] PreprocessImage(Texture2D image)
    {
        // 调整大小到 224x224
        Texture2D resized = ResizeTexture(image, 224, 224);
        
        // 转换为 float 数组,范围 [0, 1]
        float[] pixels = new float[3 * 224 * 224];
        
        // ImageNet 归一化参数
        float[] mean = { 0.485f, 0.456f, 0.406f };
        float[] std = { 0.229f, 0.224f, 0.225f };
        
        int index = 0;
        for (int c = 0; c < 3; c++)  // RGB 通道
        {
            for (int y = 0; y < 224; y++)
            {
                for (int x = 0; x < 224; x++)
                {
                    Color pixel = resized.GetPixel(x, y);
                    float value = 0;
                    
                    if (c == 0) value = pixel.r;
                    else if (c == 1) value = pixel.g;
                    else value = pixel.b;
                    
                    // 归一化
                    pixels[index++] = (value - mean[c]) / std[c];
                }
            }
        }
        
        return pixels;
    }
    
    private float[] Softmax(float[] scores)
    {
        float max = scores.Max();
        float[] exp = scores.Select(s => Mathf.Exp(s - max)).ToArray();
        float sum = exp.Sum();
        return exp.Select(e => e / sum).ToArray();
    }
    
    void OnDestroy()
    {
        session?.Dispose();
    }
}
```

### 场景 2: 语音活动检测(VAD)

```csharp
using UnityEngine;
using Microsoft.ML.OnnxRuntime;
using Microsoft.ML.OnnxRuntime.Tensors;
using System.Collections.Generic;
using System.Linq;

public class SileroVAD : MonoBehaviour
{
    private InferenceSession session;
    private const int SAMPLE_RATE = 16000;
    private const int CHUNK_SIZE = 512;
    
    // LSTM 状态 (需要在多次调用间保持)
    private float[] h;  // hidden state
    private float[] c;  // cell state
    
    void Start()
    {
        // 加载模型
        string modelPath = Application.streamingAssetsPath + "/silero_vad.onnx";
        session = new InferenceSession(modelPath);
        
        // 初始化状态
        h = new float[2 * 64];  // 2 layers * 64 units
        c = new float[2 * 64];
        
        Debug.Log("VAD 模型初始化完成");
    }
    
    public float DetectSpeech(float[] audioChunk)
    {
        // ===== 输入 1: 音频数据 =====
        var inputTensor = new DenseTensor<float>(
            audioChunk,
            new[] { 1, CHUNK_SIZE }
        );
        
        // ===== 输入 2: LSTM 隐藏状态 =====
        var hTensor = new DenseTensor<float>(h, new[] { 2, 1, 64 });
        var cTensor = new DenseTensor<float>(c, new[] { 2, 1, 64 });
        
        // ===== 输入 3: 采样率 =====
        var srTensor = new DenseTensor<long>(
            new long[] { SAMPLE_RATE },
            new[] { 1 }
        );
        
        var inputs = new List<NamedOnnxValue> {
            NamedOnnxValue.CreateFromTensor("input", inputTensor),
            NamedOnnxValue.CreateFromTensor("h", hTensor),
            NamedOnnxValue.CreateFromTensor("c", cTensor),
            NamedOnnxValue.CreateFromTensor("sr", srTensor)
        };
        
        // ===== 执行推理 =====
        using (var results = session.Run(inputs))
        {
            // ===== 输出 1: 语音概率 =====
            var probability = results
                .First(x => x.Name == "output")
                .AsEnumerable<float>()
                .First();
            
            // ===== 输出 2 & 3: 更新 LSTM 状态 =====
            h = results
                .First(x => x.Name == "hn")
                .AsEnumerable<float>()
                .ToArray();
                
            c = results
                .First(x => x.Name == "cn")
                .AsEnumerable<float>()
                .ToArray();
            
            return probability;
        }
    }
    
    // 重置状态 (比如开始新的音频流)
    public void Reset()
    {
        h = new float[2 * 64];
        c = new float[2 * 64];
    }
    
    void OnDestroy()
    {
        session?.Dispose();
    }
}

// ===== 使用示例 =====
public class VoiceRecorder : MonoBehaviour
{
    private SileroVAD vad;
    private AudioClip micClip;
    private float[] audioBuffer = new float[512];
    
    void Start()
    {
        vad = GetComponent<SileroVAD>();
        
        // 开始录音
        micClip = Microphone.Start(null, true, 10, 16000);
    }
    
    void Update()
    {
        // 获取音频数据
        int position = Microphone.GetPosition(null);
        micClip.GetData(audioBuffer, position - 512);
        
        // 检测语音
        float speechProb = vad.DetectSpeech(audioBuffer);
        
        if (speechProb > 0.5f)
        {
            Debug.Log($"检测到语音! 置信度: {speechProb:P}");
        }
    }
}
```

### 场景 3: 文本生成模型 (GPT 风格)

```csharp
using UnityEngine;
using Microsoft.ML.OnnxRuntime;
using Microsoft.ML.OnnxRuntime.Tensors;
using System.Collections.Generic;
using System.Linq;

public class TextGenerator : MonoBehaviour
{
    private InferenceSession session;
    private Dictionary<string, int> vocab;  // 词汇表
    private const int MAX_LENGTH = 128;
    
    void Start()
    {
        // 加载模型和词汇表
        session = new InferenceSession("gpt2.onnx");
        vocab = LoadVocabulary("vocab.json");
    }
    
    public string Generate(string prompt, int maxTokens = 50)
    {
        // ===== 步骤 1: 文本转 Token =====
        var tokens = Tokenize(prompt);
        
        // ===== 步骤 2: 自回归生成 =====
        for (int i = 0; i < maxTokens; i++)
        {
            // 准备输入
            var inputIds = new DenseTensor<long>(
                tokens.Select(t => (long)t).ToArray(),
                new[] { 1, tokens.Count }
            );
            
            var inputs = new List<NamedOnnxValue> {
                NamedOnnxValue.CreateFromTensor("input_ids", inputIds)
            };
            
            // 执行推理
            using (var results = session.Run(inputs))
            {
                var logits = results.First().AsTensor<float>();
                
                // 获取最后一个 token 的预测
                int vocabSize = logits.Dimensions[2];
                int lastPos = tokens.Count - 1;
                
                float[] lastLogits = new float[vocabSize];
                for (int v = 0; v < vocabSize; v++)
                {
                    lastLogits[v] = logits[0, lastPos, v];
                }
                
                // 采样下一个 token
                int nextToken = Sample(lastLogits);
                
                // 结束符检查
                if (nextToken == vocab["<|endoftext|>"])
                    break;
                
                tokens.Add(nextToken);
            }
        }
        
        // ===== 步骤 3: Token 转文本 =====
        return Detokenize(tokens);
    }
    
    private List<int> Tokenize(string text)
    {
        // 简化示例,实际需要 BPE 分词器
        return text.Split(' ')
            .Select(word => vocab.GetValueOrDefault(word, vocab["<|unk|>"]))
            .ToList();
    }
    
    private string Detokenize(List<int> tokens)
    {
        var reverseVocab = vocab.ToDictionary(kv => kv.Value, kv => kv.Key);
        return string.Join(" ", tokens.Select(t => reverseVocab[t]));
    }
    
    private int Sample(float[] logits, float temperature = 1.0f)
    {
        // Temperature 采样
        var probs = Softmax(logits.Select(l => l / temperature).ToArray());
        
        // 随机采样
        float rand = Random.value;
        float cumsum = 0;
        for (int i = 0; i < probs.Length; i++)
        {
            cumsum += probs[i];
            if (rand < cumsum)
                return i;
        }
        return probs.Length - 1;
    }
    
    void OnDestroy()
    {
        session?.Dispose();
    }
}
```

---

## 工程实践最佳实践

### 1. 资源管理

```csharp
public class ModelManager : MonoBehaviour
{
    private InferenceSession session;
    
    // ❌ 错误: 忘记释放资源
    void BadExample()
    {
        var session = new InferenceSession("model.onnx");
        // 使用 session...
        // 忘记 Dispose,导致内存泄漏!
    }
    
    // ✅ 正确: 使用 using 语句
    void GoodExample1()
    {
        using (var session = new InferenceSession("model.onnx"))
        {
            // 使用 session...
        }  // 自动释放
    }
    
    // ✅ 正确: 手动管理生命周期
    void Start()
    {
        session = new InferenceSession("model.onnx");
    }
    
    void OnDestroy()
    {
        session?.Dispose();
    }
    
    // ✅ 正确: 每次推理后释放输出
    void RunInference()
    {
        var inputs = PrepareInputs();
        
        using (var results = session.Run(inputs))
        {
            ProcessResults(results);
        }  // results 自动释放
    }
}
```

### 2. 性能优化

```csharp
public class OptimizedModel : MonoBehaviour
{
    private InferenceSession session;
    private SessionOptions options;
    
    // ===== 优化 1: 图优化 =====
    void SetupOptimizations()
    {
        options = new SessionOptions();
        
        // 启用所有优化
        options.GraphOptimizationLevel = 
            GraphOptimizationLevel.ORT_ENABLE_ALL;
        
        // 设置线程数
        options.IntraOpNumThreads = 4;  // CPU 并行
        options.InterOpNumThreads = 1;  // 图间并行
        
        session = new InferenceSession("model.onnx", options);
    }
    
    // ===== 优化 2: 复用 Tensor =====
    private DenseTensor<float> inputTensor;
    
    void Start()
    {
        // 预分配 Tensor,避免每次推理时重新创建
        inputTensor = new DenseTensor<float>(new[] { 1, 512 });
    }
    
    float RunOptimized(float[] data)
    {
        // 复用已有的 Tensor,只更新数据
        for (int i = 0; i < data.Length; i++)
        {
            inputTensor.SetValue(i, data[i]);
        }
        
        var inputs = new List<NamedOnnxValue> {
            NamedOnnxValue.CreateFromTensor("input", inputTensor)
        };
        
        using (var results = session.Run(inputs))
        {
            return results.First().AsEnumerable<float>().First();
        }
    }
    
    // ===== 优化 3: 异步推理 =====
    async System.Threading.Tasks.Task<float> RunAsync(float[] data)
    {
        return await System.Threading.Tasks.Task.Run(() =>
        {
            return RunOptimized(data);
        });
    }
}
```

### 3. 错误处理

```csharp
public class RobustModel : MonoBehaviour
{
    private InferenceSession session;
    
    bool TryLoadModel(string path)
    {
        try
        {
            session = new InferenceSession(path);
            
            // 验证模型
            ValidateModel();
            
            return true;
        }
        catch (System.IO.FileNotFoundException)
        {
            Debug.LogError($"模型文件未找到: {path}");
            return false;
        }
        catch (OnnxRuntimeException ex)
        {
            Debug.LogError($"模型加载失败: {ex.Message}");
            return false;
        }
    }
    
    void ValidateModel()
    {
        // 检查必需的输入
        if (!session.InputMetadata.ContainsKey("input"))
        {
            throw new System.Exception("模型缺少 'input' 输入");
        }
        
        // 检查输入形状
        var inputShape = session.InputMetadata["input"].Dimensions;
        if (inputShape.Length != 2)
        {
            throw new System.Exception("输入形状不正确");
        }
    }
    
    float? SafeRunInference(float[] data)
    {
        try
        {
            var tensor = new DenseTensor<float>(data, new[] { 1, data.Length });
            var inputs = new List<NamedOnnxValue> {
                NamedOnnxValue.CreateFromTensor("input", tensor)
            };
            
            using (var results = session.Run(inputs))
            {
                return results.First().AsEnumerable<float>().First();
            }
        }
        catch (OnnxRuntimeException ex)
        {
            Debug.LogError($"推理失败: {ex.Message}");
            return null;
        }
    }
}
```

---

## 调试与问题排查

### 常见问题速查表

|错误信息|可能原因|解决方法|
|---|---|---|
|`EntryPointNotFoundException`|原生 DLL 未找到或版本错误|检查 DLL 位置和架构(x64/x86)|
|`DllNotFoundException`|onnxruntime.dll 缺失|确保 DLL 在正确的 Plugins 文件夹|
|`TypeInitializationException`|P/Invoke 初始化失败|检查 DLL 依赖(VC++ Runtime)|
|`Invalid input shape`|输入 Tensor 形状错误|对比模型期望的 shape|
|`Out of memory`|模型太大或泄漏|减小 batch size,检查 Dispose|

### 调试工具箱

```csharp
public class ModelDebugger : MonoBehaviour
{
    // ===== 工具 1: 打印模型信息 =====
    public static void PrintModelInfo(InferenceSession session)
    {
        Debug.Log("===== 模型输入 =====");
        foreach (var input in session.InputMetadata)
        {
            Debug.Log($"名称: {input.Key}");
            Debug.Log($"类型: {input.Value.ElementType}");
            Debug.Log($"形状: [{string.Join(", ", input.Value.Dimensions)}]");
        }
        
        Debug.Log("\n===== 模型输出 =====");
        foreach (var output in session.OutputMetadata)
        {
            Debug.Log($"名称: {output.Key}");
            Debug.Log($"类型: {output.Value.ElementType}");
            Debug.Log($"形状: [{string.Join(", ", output.Value.Dimensions)}]");
        }
    }
    
    // ===== 工具 2: 验证输入数据 =====
    public static bool ValidateInput(
        DenseTensor<float> tensor,
        int[] expectedShape)
    {
        var actualShape = tensor.Dimensions.ToArray();
        
        if (actualShape.Length != expectedShape.Length)
        {
            Debug.LogError($"维度数量不匹配: " +
                $"期望 {expectedShape.Length}, " +
                $"实际 {actualShape.Length}");
            return false;
        }
        
        for (int i = 0; i < actualShape.Length; i++)
        {
            if (expectedShape[i] != -1 &&  // -1 表示动态维度
                actualShape[i] != expectedShape[i])
            {
                Debug.LogError($"第 {i} 维不匹配: " +
                    $"期望 {expectedShape[i]}, " +
                    $"实际 {actualShape[i]}");
                return false;
            }
        }
        
        // 检查数据范围
        var data = tensor.ToArray();
        float min = data.Min();
        float max = data.Max();
        Debug.Log($"数据范围: [{min}, {max}]");
        
        // 检查异常值
        if (float.IsNaN(min) || float.IsInfinity(max))
        {
            Debug.LogWarning("输入包含 NaN 或 Infinity!");
            return false;
        }
        
        return true;
    }
    
    // ===== 工具 3: 性能分析 =====
    public static float MeasureInference(
        InferenceSession session,
        List<NamedOnnxValue> inputs,
        int iterations = 100)
    {
        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        
        for (int i = 0; i < iterations; i++)
        {
            using (var results = session.Run(inputs))
            {
                // 强制读取结果确保计算完成
                var _ = results.First().AsEnumerable<float>().First();
            }
        }
        
        stopwatch.Stop();
        float avgMs = stopwatch.ElapsedMilliseconds / (float)iterations;
        
        Debug.Log($"平均推理时间: {avgMs:F2} ms ({1000/avgMs:F1} FPS)");
        return avgMs;
    }
    
    // ===== 工具 4: 保存/加载测试数据 =====
    public static void SaveTestData(
        string path,
        float[] data,
        int[] shape)
    {
        var json = new {
            shape = shape,
            data = data
        };
        
        string jsonStr = JsonUtility.ToJson(json);
        System.IO.File.WriteAllText(path, jsonStr);
    }
}
```

### 使用示例

```csharp
public class Example : MonoBehaviour
{
    void Start()
    {
        var session = new InferenceSession("model.onnx");
        
        // 1. 打印模型结构
        ModelDebugger.PrintModelInfo(session);
        
        // 2. 准备测试输入
        var testInput = new DenseTensor<float>(new[] { 1, 512 });
        
        // 3. 验证输入
        if (!ModelDebugger.ValidateInput(testInput, new[] { 1, 512 }))
        {
            Debug.LogError("输入验证失败!");
            return;
        }
        
        // 4. 性能测试
        var inputs = new List<NamedOnnxValue> {
            NamedOnnxValue.CreateFromTensor("input", testInput)
        };
        ModelDebugger.MeasureInference(session, inputs);
    }
}
```

---

## 总结与最佳实践清单

### ✅ 快速接入检查清单

- [ ] **环境准备**
    
    - [ ] Unity 版本 >= 2021.2
    - [ ] 安装正确版本的 onnxruntime.dll (x64)
    - [ ] 安装 Microsoft.ML.OnnxRuntime.dll
    - [ ] VC++ Runtime 已安装
- [ ] **模型加载**
    
    - [ ] 模型文件放在 StreamingAssets 或可访问路径
    - [ ] 使用 SessionOptions 配置优化
    - [ ] 在 Start() 中加载,避免每帧加载
    - [ ] 在 OnDestroy() 中释放资源
- [ ] **数据处理**
    
    - [ ] 理解模型的输入/输出 shape
    - [ ] 正确预处理数据(归一化、resize 等)
    - [ ] 使用 ValidateInput 检查数据
    - [ ] 注意数据类型匹配(float/long/int)
- [ ] **推理执行**
    
    - [ ] 使用 using 语句管理 results
    - [ ] 考虑异步执行避免卡顿
    - [ ] 复用 Tensor 提高性能
    - [ ] 添加异常处理
- [ ] **结果处理**
    
    - [ ] 正确解析输出 Tensor
    - [ ] 应用后处理(Softmax、NMS 等)
    - [ ] 转换为游戏可用格式

### 🚀 性能优化建议

1. **模型优化**: 使用 GraphOptimizationLevel.ORT_ENABLE_ALL
2. **内存优化**: 预分配 Tensor,避免频繁 GC
3. **并行优化**: 合理设置线程数
4. **批处理**: 增大 batch size 提高吞吐量(如适用)
5. **量化**: 使用 INT8 量化模型减小大小和加速

### 📚 学习资源

- **ONNX Runtime 官方文档**: https://onnxruntime.ai/docs/
- **ONNX 模型库**: https://github.com/onnx/models
- **Unity ML-Agents**: 学习 Unity 中的 ML 最佳实践

---

**完成这份指南后,你应该能够:**

- ✅ 理解 ONNX Runtime 的工作原理
- ✅ 正确加载和运行 ONNX 模型
- ✅ 处理各种输入/输出格式
- ✅ 调试和优化模型性能
- ✅ 在 Unity 项目中集成 AI 模型