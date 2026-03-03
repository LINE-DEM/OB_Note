# 计划：OVRLipSync → uLipSync 迁移（WebGL 嘴型驱动交付方案）

## 背景

当前项目使用 `OVRLipSync.dll`（Win64 原生库）驱动角色嘴型，无法在 WebGL 平台运行。 需交付一套基于 **uLipSync**（纯C#，支持WebGL）的等效方案，保持相同的角色模型、BlendShape 索引和业务接口。

**约束确认：**

- 同一角色模型 → BlendShape 索引完全沿用
- 音频来源：AudioClip 文件（AudioSource 播放）
- 校准策略：交付时提供针对 TTS 中文语音的预制 Profile

---

## 关键文件

|文件|用途|
|---|---|
|`Assets/AIChatTookit/LipSync/Scripts/AudioToLipSample.cs`|被替代的核心脚本（接口参考）|
|`Assets/AIChatTookit/LipSync/Scripts/OVRLipSync.cs`|旧桥接层，迁移后废弃|
|`Assets/c#/SystemLayer/Audio/AudioMgr.cs`|唯一调用方，需最小修改|
|`Assets/AIChatTookit/LipSync/Prefabs/LipSyncInterface.prefab`|参考配置|
|`Packages/manifest.json`|添加 uLipSync 包|

**新建文件：**

- `Assets/AIChatTookit/LipSync/Scripts/ULipSyncController.cs`
- `Assets/AIChatTookit/LipSync/Profiles/uLipSync-Profile-ChineseTTS.asset`
- `Assets/AIChatTookit/LipSync/Prefabs/LipSync_WebGL.prefab`

---

## 实施步骤

### Step 1：安装 uLipSync 包

在 `Packages/manifest.json` 的 `dependencies` 中添加：

```json
"com.hecomi.ulipsync": "https://github.com/hecomi/uLipSync.git#upm"
```

或通过 Unity Package Manager → Add package from git URL 输入同一地址。

---

### Step 2：创建 uLipSync Profile（中文 TTS 校准）

**2.1 在 Project 窗口创建：** 右键 → `Create > uLipSync > Profile`，命名为 `uLipSync-Profile-ChineseTTS`，存放于 `Assets/AIChatTookit/LipSync/Profiles/`

**2.2 在 Profile 中按顺序注册 14 个音素（名称必须与脚本中一致）：**

|Profile 中的音素名|对应中文 TTS 发音|校准音|
|---|---|---|
|`A`|元音 a|"啊——"|
|`I`|元音 i|"衣——"|
|`U`|元音 u|"呜——"|
|`E`|元音 e|"诶——"|
|`O`|元音 o|"哦——"|
|`PP`|b/p 爆破音|"爸爸、普通"|
|`FF`|f 唇齿音|"发现、飞机"|
|`TH`|（中文无此音）|**录 1 秒静音**|
|`DD`|d/t 舌尖音|"大家、太好"|
|`kk`|k/g 软腭音|"可以、感觉"|
|`CH`|zh/ch/sh 卷舌音|"知道、出来"|
|`SS`|s/z/x 齿音|"所以、谢谢"|
|`nn`|n/m 鼻音|"那么、明白"|
|`RR`|r 卷舌近音|"然后、日"|

**2.3 校准操作流程（在 Editor 中）：**

1. 创建临时场景，新建 GameObject 挂载：`AudioSource` + `uLipSync` + `uLipSyncCalibrationAudioPlayer`
2. `uLipSync` 的 Profile 字段指向 `uLipSync-Profile-ChineseTTS`
3. `uLipSyncCalibrationAudioPlayer` 的 AudioClip 指向对应校准音频，循环播放
4. 点 Play，在 `uLipSync` 组件的 Calibration 区域找到对应音素行，按住 `Calib` 按钮约 1.5 秒
5. 所有 14 个音素依次完成后，`Ctrl+S` 保存 ScriptableObject

---

### Step 3：创建 `ULipSyncController.cs`

新建 `Assets/AIChatTookit/LipSync/Scripts/ULipSyncController.cs`，核心设计要点：

**字段（与 AudioToLipSample 公开接口兼容）：**

```csharp
public SkinnedMeshRenderer meshRenderer;
public SkinnedMeshRenderer meshRenderer_Teeth;
public SkinnedMeshRenderer meshRenderer_tongue;
public VisemeBlenderShapeIndexMap m_VisemeIndex;
public float blendWeightMultiplier = 65f;
public float maxBlendWeight = 70f;
public uLipSync.uLipSync uLipSyncComponent;  // Inspector 可配置或自动 GetComponent
[Range(0f, 0.1f)]
public float silenceThreshold = 0.005f;
```

**生命周期：**

```csharp
void Awake() {
    BuildPhonemeMap();  // 构建 音素名→(bsIndex, Queue) 查找表
    if (uLipSyncComponent == null)
        uLipSyncComponent = GetComponent<uLipSync.uLipSync>();
    uLipSyncComponent.onLipSyncUpdate.AddListener(OnLipSyncUpdate);
}
void OnDestroy() => uLipSyncComponent?.onLipSyncUpdate.RemoveListener(OnLipSyncUpdate);
void OnDisable() => ClearAllQueues();  // 禁用时清空平滑队列
```

**核心回调（替代原 OnAudioFilterRead + OVRLipSync.ProcessFrame）：**

```csharp
// 注意：安装 uLipSync 后确认 LipSyncInfo 的实际字段名
// 访问所有音素权重可能是 info.phonemeRatios 或 uLipSyncComponent.phonemeRatios
// 具体以安装后的 API 为准
public void OnLipSyncUpdate(LipSyncInfo info) {
    if (info.volume < silenceThreshold) return;

    // 遍历所有音素权重
    foreach (var kv in info.phonemeRatios) {   // 或 uLipSyncComponent.phonemeRatios
        string phoneme = kv.Key;
        float ratio = kv.Value;  // 0~1

        if (!_phonemeMap.TryGetValue(phoneme, out var mapping) || mapping.bsIndex == 999)
            continue;

        float weight = Mathf.Min(ratio * blendWeightMultiplier, maxBlendWeight);
        float smoothed = PushAndAverage(mapping.queue, weight);

        if (smoothed <= 0f) continue;

        meshRenderer?.SetBlendShapeWeight(mapping.bsIndex, smoothed);
        meshRenderer_Teeth?.SetBlendShapeWeight(mapping.bsIndex, smoothed);

        int tongueIdx = GetTongueIndex(mapping.bsIndex);
        if (tongueIdx != 999)
            meshRenderer_tongue?.SetBlendShapeWeight(tongueIdx, smoothed);
    }
}
```

**音素查找表（BuildPhonemeMap）：**

```csharp
_phonemeMap = new Dictionary<string, (int, Queue<float>)> {
    { "A",  (m_VisemeIndex.A,  _aQueue)  },
    { "I",  (m_VisemeIndex.I,  _iQueue)  },
    { "U",  (m_VisemeIndex.U,  _uQueue)  },
    { "E",  (m_VisemeIndex.E,  _eQueue)  },
    { "O",  (m_VisemeIndex.O,  _oQueue)  },
    { "PP", (m_VisemeIndex.PP, _ppQueue) },
    { "FF", (m_VisemeIndex.FF, _ffQueue) },
    { "TH", (m_VisemeIndex.TH, _thQueue) },
    { "DD", (m_VisemeIndex.DD, _ddQueue) },
    { "kk", (m_VisemeIndex.kk, _kkQueue) },  // 大小写必须与 Profile 一致
    { "CH", (m_VisemeIndex.CH, _chQueue) },
    { "SS", (m_VisemeIndex.SS, _ssQueue) },
    { "nn", (m_VisemeIndex.nn, _nnQueue) },  // 同上
    { "RR", (m_VisemeIndex.RR, _rrQueue) },
    { "-",  (999, new Queue<float>())    },  // uLipSync 的静音标记
};
```

**平滑系统（与 AudioToLipSample 完全一致，smoothCount=5）：**

```csharp
// 14 个独立 Queue<float>，每个音素独立平滑
private float PushAndAverage(Queue<float> q, float v) {
    if (q.Count >= 5) q.Dequeue();
    q.Enqueue(v);
    if (q.Count < 5) return 0f;
    float sum = 0; foreach (float f in q) sum += f;
    return sum / q.Count;
}
```

**保留接口（AudioMgr 调用兼容）：**

```csharp
public void InitAllBlendShape() {
    // 与原 AudioToLipSample 完全相同的实现
    // 清零三个 SkinnedMeshRenderer 的所有 BlendShape + 清空所有队列
}
```

**舌头映射（GetTongueIndex，与原代码完全相同）：**

```csharp
private int GetTongueIndex(int bsIndex) {
    if (bsIndex == m_VisemeIndex.E)  return 0;
    if (bsIndex == m_VisemeIndex.DD) return 1;
    if (bsIndex == m_VisemeIndex.kk) return 2;
    if (bsIndex == m_VisemeIndex.CH) return 3;
    if (bsIndex == m_VisemeIndex.I)  return 4;
    if (bsIndex == m_VisemeIndex.SS) return 5;
    if (bsIndex == m_VisemeIndex.FF) return 5;
    if (bsIndex == m_VisemeIndex.A)  return 6;
    return 999;
}
```

---

### Step 4：修改 AudioMgr.cs（最小改动，2处）

```csharp
// 字段类型替换（约第16行）
// 改前：public AudioToLipSample audioToLipSample;
public ULipSyncController audioToLipSample;

// BindPlayerContext 中的 GetComponentCached 调用
// 改前：GetComponentCached<AudioToLipSample>()
GetComponentCached<ULipSyncController>()
```

其余所有调用（`.enabled`、`.InitAllBlendShape()`）无需改动。

---

### Step 5：配置 Prefab

**GameObject 层级：**

```
[LipSyncNode] (新建)
├── AudioSource
├── uLipSync              ← Profile 指向 uLipSync-Profile-ChineseTTS
├── uLipSyncAudioSource   ← WebGL 必须！桥接 AudioClip.GetData()
└── ULipSyncController    ← Inspector 配置 meshRenderer/索引/参数
```

**uLipSync 组件关键配置：**

- Profile → `uLipSync-Profile-ChineseTTS`
- onLipSyncUpdate → 绑定 `ULipSyncController.OnLipSyncUpdate`（或代码 AddListener 二选一）

**ULipSyncController Inspector 配置（沿用原值）：**

- blendWeightMultiplier = 65
- maxBlendWeight = 70
- m_VisemeIndex 各字段值与原 prefab 完全相同

---

### Step 6：处理旧文件

|文件|处理|
|---|---|
|`OVRLipSync.dll`|Plugin Import Settings 中取消所有平台勾选（不删除避免 GUID 报错）|
|`OVRLipSync.cs` / `AudioToLipSample.cs`|移至 `_Deprecated/` 文件夹|

---

## 交付物清单

```
Assets/AIChatTookit/LipSync/
├── Scripts/ULipSyncController.cs
├── Profiles/uLipSync-Profile-ChineseTTS.asset
└── Prefabs/LipSync_WebGL.prefab

Packages/manifest.json  (含 uLipSync Git URL 条目)
Assets/c#/SystemLayer/Audio/AudioMgr.cs  (2处字段类型修改)
```

---

## 关键风险提示

1. **音素名称大小写**：Profile 中的名称（kk、nn）必须与 `BuildPhonemeMap` 中的键完全一致
2. **uLipSync API 确认**：安装包后确认 `LipSyncInfo.phonemeRatios` 的实际字段名（不同版本可能有差异，可能需改为 `uLipSyncComponent.phonemeRatios`）
3. **WebGL AudioClip 加载**：确保 AudioClip 的 Load Type 为 `Decompress On Load`（非流式），避免 `GetData()` 返回空
4. **禁止 PlayOneShot**：WebGL 上 uLipSync 无法 hook PlayOneShot 的音频数据，必须用 `clip + Play()`（当前 AudioMgr 已符合此要求）

## 验证方式

1. Editor 中：播放任意中文 TTS 音频，观察 ULipSyncController Inspector 的 blendShape 数值是否随音频变化
2. WebGL Build：在浏览器中播放音频，观察角色嘴型是否同步运动
3. 对比测试：同一段音频在原方案和新方案下，分别截图5个关键帧比较嘴型覆盖率