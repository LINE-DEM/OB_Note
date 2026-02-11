---
创建时间: 2026-02-11 17:49
复习次数: 1
---

# MainUIForm 重构指南

  

> 生成日期：2026-02-11

> 目标：引入集中式状态机，解决状态散落、显隐冲突、难以调试的问题，并支持"导览时替换语言按钮"新需求。

  

---

  

## 一、现状问题诊断

  

### 1.1 mainState 写入/读取全景

  

**写入端（4处，全部在 MainUIForm.cs 内）：**

  

| 行号 | 方法 | 赋值 | 触发场景 |

|------|------|------|----------|

| 33 | 字段初始化 | `QiPaoMain` | 对象创建 |

| 267 | `QiPaoToMainForm()` | → `HuiFu` | 点击气泡 / 录音进入问答 |

| 344 | `OnClickGoBack()` | → `QiPaoMain` | 返回气泡页 |

| 610 | `RecordBtnDown()` | → `HuiFu` | 录音按钮按下 |

  

**读取端（21+ 处，散布 13 个文件）：**

  

| 文件 | 读取次数 | 判断目的 |

|------|----------|----------|

| MainUIForm.cs | 9 | Update轮询 / 计时器守卫 / 状态切换前置检查 |

| BubbleManager.cs | 4 | 气泡渲染策略（最大消费者） |

| GuideUIForm.cs | 2 | 导览介绍流程控制 |

| GuideMgr.cs | 2 | 导览管理流程控制 |

| Record.cs | 1 | 录音行为决策 |

| XunFeiManager.cs | 1 | 语音处理决策 |

| AudioMgr.cs | 1 | 音频播放决策 |

| QiPaoUIForm.cs | 1 | 气泡交互决策 |

| InputMonitor.cs | 1 | 输入监听决策 |

| InputMonitor1.cs | 1 | 输入监听决策 |

| InputMonitor2.cs | 1 | 输入监听决策 |

| LogonUIForm.cs | 1 | 登录跳转决策 |

| AAAAAAAAA.cs | 1 | 网络行为决策 |

  

**核心问题**：所有外部系统通过 `UIManager.GetInstance().GetUIForm<MainUIForm>().mainState` 轮询读取，形成隐式耦合。无法通过断点追踪"谁在什么时机因为读到了什么值而做出了错误决策"。

  

---

  

### 1.2 散落的 bool 状态组合

  

实际影响系统行为的决策因子有 7 个，分布在 5 个不同的单例中：

  

```

MainUIForm.mainState          (QiPaoMain / HuiFu)    ← MainUIForm

MainUIForm.isAutoDaoLan       (true / false)          ← MainUIForm

NetWorkGuide.isMoving          (true / false)          ← NetWorkGuide

NetWorkGuide.isTrueMoving      (true / false)          ← NetWorkGuide

GuideUIForm 是否打开            (open / closed)         ← UIManager

GuideMgr.isOutingChargePoint   (true / false)          ← GuideMgr

Record._isRecording            (true / false)          ← Record

```

  

这些 bool 的组合构成了系统真正的"阶段"，但没有任何地方将它们统一建模。

  

---

  

### 1.3 LanguageChange 显隐控制的冲突

  

当前有 5 个方法、5 个不同的调用源控制显隐，只用 `_isLanguageChangeHidden` 一个 bool 做仲裁：

  

```

调用源                               方法                        动作

─────────────────────────────────────────────────────────────────────

QiPaoToMainForm()                → FadeOutLanguageChangeAndDaoLans()  → 隐藏

GuideUIForm.Display()            → HideLanguageChange()               → 隐藏

QiPaoUIForm.HideQiPao()          → HideLanguageChange()               → 隐藏

OnClickDaoLan() 切自动            → HideLanguageChange()               → 隐藏

Update() 移动守卫                 → HideLanguageChange()               → 隐藏（每帧！）

  

GuideUIForm.Hiding()             → ShowLanguageChange()               → 显示

QiPaoUIForm.DoMoveQiPaoBack()    → FadeInLanguageChangeAndDaoLans()   → 显示（无条件!）

GuideMgr 充电桩退出               → ShowLanguageChange(true)           → 显示（绕过移动检查!）

OnClickDaoLan() 切手动            → ShowLanguageChange()               → 显示

ShowGuideHandUI()                → ShowLanguageChange()               → 显示

```

  

**冲突场景举例**：

- 机器人正在移动 → Update 每帧 Hide

- 此时 GuideUIForm.Hiding() 被调用 → Show，下一帧又被 Update Hide

- QiPaoUIForm.DoMoveQiPaoBack() → 无条件 FadeIn，不检查移动/导览状态

  

---

  

## 二、目标架构

  

### 2.1 引入 AppStateManager（集中式状态机）

  

```

                    ┌─────────────────────┐

                    │   AppStateManager   │

                    │   (唯一状态权威)      │

                    ├─────────────────────┤

                    │ Current: AppState   │

                    │ Previous: AppState  │

                    │ GuideMode: bool     │

                    ├─────────────────────┤

                    │ TransitionTo()      │

                    │ OnStateChanged ─────┼──→ 事件广播

                    └─────────┬───────────┘

                              │

              ┌───────────────┼───────────────┐

              │               │               │

         ┌────▼───┐    ┌─────▼────┐   ┌──────▼──────┐

         │MainUI  │    │BubbleMgr │   │ AudioMgr    │

         │Form    │    │          │   │ Record      │

         │        │    │          │   │ XunFeiMgr   │

         └────────┘    └──────────┘   └─────────────┘

         订阅事件         订阅事件        订阅事件

         更新UI显隐       更新气泡策略     更新音频/录音策略

```

  

### 2.2 状态定义

  

```csharp

/// <summary>

/// 文件路径建议: Assets/c#/State/AppStateManager.cs

/// 全局应用状态机，取代散落各处的 bool 组合判断

/// </summary>

public class AppStateManager : MonoBehaviour

{

    public static AppStateManager Instance { get; private set; }

  

    /// <summary>

    /// 主状态：互斥，同一时刻只能处于一种

    /// </summary>

    public enum AppState

    {

        Idle,       // 气泡页待机（原 QiPaoMain）

        Talking,    // 问答页对话中（原 HuiFu）

        Guiding,    // 导览介绍播放中

        Moving,     // 机器人移动中

    }

  

    /// <summary>

    /// 导览模式：正交维度，与主状态独立

    /// </summary>

    public enum GuideMode

    {

        None,       // 未在导览

        Auto,       // 自动导览

        Manual,     // 手动导览

    }

  

    // ---- 状态 ----

    public AppState Current { get; private set; } = AppState.Idle;

    public AppState Previous { get; private set; } = AppState.Idle;

    public GuideMode CurrentGuideMode { get; private set; } = GuideMode.None;

  

    // ---- 事件 ----

    public event Action<AppState, AppState> OnStateChanged;          // old, new

    public event Action<GuideMode, GuideMode> OnGuideModeChanged;    // old, new

  

    void Awake()

    {

        if (Instance != null && Instance != this) { Destroy(gameObject); return; }

        Instance = this;

    }

  

    /// <summary>

    /// 唯一的主状态变更入口。所有状态切换必须经过此方法。

    /// Debug 时在此处加断点即可追踪一切状态变更。

    /// </summary>

    public void TransitionTo(AppState newState)

    {

        if (Current == newState) return;

  

        Previous = Current;

        Current = newState;

  

        Debug.Log($"[AppState] {Previous} → {Current}  (frame {Time.frameCount})");

        OnStateChanged?.Invoke(Previous, Current);

    }

  

    /// <summary>

    /// 导览模式变更入口

    /// </summary>

    public void SetGuideMode(GuideMode mode)

    {

        if (CurrentGuideMode == mode) return;

  

        var old = CurrentGuideMode;

        CurrentGuideMode = mode;

  

        Debug.Log($"[GuideMode] {old} → {mode}  (frame {Time.frameCount})");

        OnGuideModeChanged?.Invoke(old, mode);

    }

}

```

  

**为什么主状态和导览模式分开？**

  

因为它们是正交维度。机器人可以处于"Idle + 自动导览"，也可以"Idle + 无导览"。如果混在一个枚举里会产生组合爆炸（IdleAutoGuide / IdleManualGuide / TalkingAutoGuide ...）。

  

---

  

## 三、迁移步骤（共三个阶段）

  

### 阶段一：最小改动，支撑新需求（导览时替换语言按钮）

  

**改动范围**：新增 1 个文件，修改 2 个文件。不删除任何旧代码。

  

#### 步骤 1.1：创建 AppStateManager.cs

  

文件路径：`Assets/c#/State/AppStateManager.cs`

  

内容如上方 2.2 节所示。挂载到场景中一个持久 GameObject 上（或使用 DontDestroyOnLoad）。

  

#### 步骤 1.2：在关键位置发出状态变更

  

```csharp

// ---- MainUIForm.cs ----

  

// QiPaoToMainForm() 中，mainState = MainState.HuiFu 之后

AppStateManager.Instance.TransitionTo(AppStateManager.AppState.Talking);

  

// OnClickGoBack() 中，mainState = MainState.QiPaoMain 之后

AppStateManager.Instance.TransitionTo(AppStateManager.AppState.Idle);

  

// ---- GuideUIForm.cs ----

  

// Display() 中

AppStateManager.Instance.TransitionTo(AppStateManager.AppState.Guiding);

  

// Hiding() / CloseGuideUI() 中

// 根据 Previous 回到之前的状态

AppStateManager.Instance.TransitionTo(AppStateManager.Instance.Previous);

  

// ---- MainUIForm.cs (导览模式切换) ----

  

// OnClickDaoLan() 切自动

AppStateManager.Instance.SetGuideMode(AppStateManager.GuideMode.Auto);

  

// OnClickDaoLan() 切手动

AppStateManager.Instance.SetGuideMode(AppStateManager.GuideMode.Manual);

```

  

#### 步骤 1.3：MainUIForm 订阅事件，实现新需求

  

```csharp

// ---- MainUIForm.cs ----

  

public GameObject newGuideButton;  // Inspector 绑定新按钮

  

void Awake()

{

    // ...existing code...

    AppStateManager.Instance.OnStateChanged += OnAppStateChanged;

    AppStateManager.Instance.OnGuideModeChanged += OnGuideModeChanged;

}

  

void OnDestroy()

{

    if (AppStateManager.Instance != null)

    {

        AppStateManager.Instance.OnStateChanged -= OnAppStateChanged;

        AppStateManager.Instance.OnGuideModeChanged -= OnGuideModeChanged;

    }

}

  

/// <summary>

/// 集中式 UI 显隐决策 — 取代所有 Hide/Show/FadeIn/FadeOut 的交叉调用

/// </summary>

void OnAppStateChanged(AppStateManager.AppState oldState, AppStateManager.AppState newState)

{

    switch (newState)

    {

        case AppStateManager.AppState.Idle:

            // 气泡页：根据导览模式决定显示哪个按钮

            if (AppStateManager.Instance.CurrentGuideMode != AppStateManager.GuideMode.None)

            {

                LanguageChange.SetActive(false);

                newGuideButton.SetActive(true);

            }

            else

            {

                LanguageChange.SetActive(true);

                newGuideButton.SetActive(false);

            }

            daoLans.SetActive(true);

            RestoreDaoLansPosition();

            RestoreCanvasGroupAlpha(LanguageChange);

            RestoreCanvasGroupAlpha(daoLans);

            break;

  

        case AppStateManager.AppState.Talking:

            // 问答页：全部隐藏

            LanguageChange.SetActive(false);

            newGuideButton.SetActive(false);

            daoLans.SetActive(false);

            break;

  

        case AppStateManager.AppState.Guiding:

            // 导览介绍中：语言按钮隐藏，显示新按钮

            LanguageChange.SetActive(false);

            newGuideButton.SetActive(true);

            daoLans.SetActive(true);

            ShiftDaoLansCompact();  // 右移补位

            break;

  

        case AppStateManager.AppState.Moving:

            // 移动中：全部隐藏

            LanguageChange.SetActive(false);

            newGuideButton.SetActive(false);

            // daoLans 保留可见但右移

            ShiftDaoLansCompact();

            break;

    }

}

  

void OnGuideModeChanged(AppStateManager.GuideMode oldMode, AppStateManager.GuideMode newMode)

{

    // 导览模式变化时，如果当前是 Idle 状态，刷新按钮显示

    if (AppStateManager.Instance.Current == AppStateManager.AppState.Idle)

    {

        if (newMode != AppStateManager.GuideMode.None)

        {

            LanguageChange.SetActive(false);

            newGuideButton.SetActive(true);

        }

        else

        {

            LanguageChange.SetActive(true);

            newGuideButton.SetActive(false);

        }

    }

}

  

// ---- 辅助方法 ----

  

void RestoreDaoLansPosition()

{

    if (daoLans == null) return;

    var rt = daoLans.GetComponent<RectTransform>();

    if (rt != null) rt.anchoredPosition = Vector2.zero;

}

  

void ShiftDaoLansCompact()

{

    if (daoLans == null) return;

    var rt = daoLans.GetComponent<RectTransform>();

    if (rt != null) rt.anchoredPosition = new Vector2(-124.11f, 0f);

}

  

void RestoreCanvasGroupAlpha(GameObject obj)

{

    if (obj == null) return;

    var cg = obj.GetComponent<CanvasGroup>();

    if (cg != null) cg.alpha = 1f;

}

```

  

#### 步骤 1.4：删除 Update 中的移动守卫

  

```csharp

// 删除 MainUIForm.Update() 中的这段（第 233-241 行）：

// if(NetWorkGuide.Instance.isTrueMoving)

// {

//     if(GuideMgr.Instance.isOutingChargePoint) { return; }

//     HideLanguageChange();

// }

//

// 改为：在 NetWorkGuide 开始/停止移动时调用：

// AppStateManager.Instance.TransitionTo(AppState.Moving);

// AppStateManager.Instance.TransitionTo(AppState.Idle);

```

  

> **阶段一完成后的效果**：新需求已实现，LanguageChange 显隐由事件驱动，不再有交叉调用冲突。`mainState` 暂时保留，老代码不受影响。

  

---

  

### 阶段二：消除 mainState 对外暴露

  

**目标**：外部系统不再直接读取 `mainUIForm.mainState`，改为订阅 `AppStateManager.OnStateChanged` 或查询 `AppStateManager.Instance.Current`。

  

#### 步骤 2.1：mainState 改为 private，通过 AppStateManager 查询

  

```csharp

// MainUIForm.cs

// 原：public MainState mainState = MainState.QiPaoMain;

// 改：

private MainState mainState = MainState.QiPaoMain;

```

  

#### 步骤 2.2：逐文件迁移读取点

  

按依赖密度从高到低排序迁移（改动量大的先改，边际收益最高）：

  

**优先级 1 — BubbleManager.cs（4 处读取）**

  

```csharp

// 原：

// mainUIForm.mainState == MainUIForm.MainState.HuiFu

// 改为：

// AppStateManager.Instance.Current == AppStateManager.AppState.Talking

  

// 更好的方式：订阅事件，缓存状态

void OnEnable()

{

    AppStateManager.Instance.OnStateChanged += OnAppStateChanged;

}

void OnDisable()

{

    AppStateManager.Instance.OnStateChanged -= OnAppStateChanged;

}

void OnAppStateChanged(AppStateManager.AppState old, AppStateManager.AppState current)

{

    isInHuiFuPanel = (current == AppStateManager.AppState.Talking);

    // ...其他逻辑

}

```

  

**优先级 2 — GuideUIForm.cs（2 处读取）**

  

```csharp

// 原：mainUIForm.mainState == MainUIForm.MainState.HuiFu

// 改：AppStateManager.Instance.Current == AppStateManager.AppState.Talking

```

  

**优先级 3 — GuideMgr.cs（2 处读取）**

  

```csharp

// 同上替换模式

```

  

**优先级 4 — 其余 8 个文件（各 1 处读取）**

  

```

Record.cs           → 替换 1 处

XunFeiManager.cs    → 替换 1 处

AudioMgr.cs         → 替换 1 处

QiPaoUIForm.cs      → 替换 1 处

InputMonitor.cs     → 替换 1 处

InputMonitor1.cs    → 替换 1 处

InputMonitor2.cs    → 替换 1 处

LogonUIForm.cs      → 替换 1 处

AAAAAAAAA.cs        → 替换 1 处

```

  

#### 步骤 2.3：验证与清理

  

- [ ] 全局搜索 `mainState` 确认无外部引用

- [ ] 全局搜索 `MainState.HuiFu` 和 `MainState.QiPaoMain` 确认无遗漏

- [ ] `MainUIForm.MainState` 枚举可标记 `[Obsolete]` 或直接删除

  

---

  

### 阶段三：消除散落的 bool，完成状态收编

  

**目标**：将 `isAutoDaoLan`、`isMoving`、`isTrueMoving` 等关键 bool 收编进状态机。

  

#### 步骤 3.1：移动状态收编

  

```csharp

// 原：NetWorkGuide 中散落的 isMoving / isTrueMoving

// 改：在 NetWorkGuide.Move() 中：

AppStateManager.Instance.TransitionTo(AppStateManager.AppState.Moving);

  

// 在 NetWorkGuide 到达目标点时：

AppStateManager.Instance.TransitionTo(AppStateManager.AppState.Idle);

  

// 其余系统查询：

// 原：NetWorkGuide.Instance.isTrueMoving

// 改：AppStateManager.Instance.Current == AppStateManager.AppState.Moving

```

  

#### 步骤 3.2：导览模式收编

  

```csharp

// 原：MainUIForm.isAutoDaoLan

// 已在阶段一通过 AppStateManager.GuideMode 替代

// 此步骤：删除 isAutoDaoLan 字段，全局替换为 AppStateManager.Instance.CurrentGuideMode

```

  

#### 步骤 3.3：删除旧的 LanguageChange 控制方法

  

以下方法在阶段三完成后可安全删除：

  

```

MainUIForm.HideLanguageChange()

MainUIForm.ShowLanguageChange()

MainUIForm.FadeOutLanguageChangeAndDaoLans()

MainUIForm.FadeInLanguageChangeAndDaoLans()

MainUIForm.KillLanguageChangeAnimation()

MainUIForm._isLanguageChangeHidden 字段

```

  

所有显隐逻辑已统一到 `OnAppStateChanged()` 事件处理器中。

  

---

  

## 四、迁移检查清单

  

### 阶段一检查清单

  

- [ ] 创建 `Assets/c#/State/AppStateManager.cs`

- [ ] 场景中挂载 AppStateManager 到持久 GameObject

- [ ] MainUIForm.Awake() 中订阅 OnStateChanged / OnGuideModeChanged

- [ ] MainUIForm.OnDestroy() 中取消订阅

- [ ] QiPaoToMainForm() 中添加 `TransitionTo(Talking)`

- [ ] OnClickGoBack() 中添加 `TransitionTo(Idle)`

- [ ] GuideUIForm.Display() 中添加 `TransitionTo(Guiding)`

- [ ] GuideUIForm.Hiding() 中添加 `TransitionTo(Previous)`

- [ ] OnClickDaoLan() 中添加 `SetGuideMode(Auto/Manual)`

- [ ] 实现 OnAppStateChanged() 集中显隐决策

- [ ] 实现 OnGuideModeChanged() 按钮切换逻辑

- [ ] 添加 newGuideButton 字段并在 Inspector 绑定

- [ ] 删除 Update 中的移动守卫（第 233-241 行）

- [ ] 在 NetWorkGuide 移动开始/结束处添加 TransitionTo

- [ ] 测试：气泡页 → 问答页 → 返回，按钮显隐正确

- [ ] 测试：导览介绍播放中，新按钮显示，语言按钮隐藏

- [ ] 测试：自动/手动导览切换，按钮状态正确

- [ ] 测试：移动中，按钮全部隐藏

  

### 阶段二检查清单

  

- [ ] mainState 改为 private

- [ ] 迁移 BubbleManager.cs（4 处）

- [ ] 迁移 GuideUIForm.cs（2 处）

- [ ] 迁移 GuideMgr.cs（2 处）

- [ ] 迁移 Record.cs（1 处）

- [ ] 迁移 XunFeiManager.cs（1 处）

- [ ] 迁移 AudioMgr.cs（1 处）

- [ ] 迁移 QiPaoUIForm.cs（1 处）

- [ ] 迁移 InputMonitor.cs / 1 / 2（各 1 处）

- [ ] 迁移 LogonUIForm.cs（1 处）

- [ ] 迁移 AAAAAAAAA.cs（1 处）

- [ ] 全局搜索确认无 mainState 外部引用残留

- [ ] 删除或标记 `[Obsolete]` MainState 枚举

  

### 阶段三检查清单

  

- [ ] NetWorkGuide.isMoving / isTrueMoving → AppState.Moving

- [ ] MainUIForm.isAutoDaoLan → AppStateManager.GuideMode

- [ ] 删除 HideLanguageChange / ShowLanguageChange / FadeIn / FadeOut

- [ ] 删除 _isLanguageChangeHidden

- [ ] 删除 KillLanguageChangeAnimation

- [ ] 全局搜索确认无 HideLanguageChange / ShowLanguageChange 残留调用

  

---

  

## 五、架构决策记录 (ADR)

  

### ADR-001：为什么用事件驱动而非继续轮询？

  

**现状**：13 个文件通过 `GetUIForm<MainUIForm>().mainState` 在 Update/协程中轮询状态。

  

**问题**：

- 调用时序不可控，同一帧内不同系统可能读到变更前/后的值

- 无法追踪"谁因为什么值做出了什么决策"

- 新增状态时需要逐个文件排查所有 if 分支

  

**决策**：采用 `OnStateChanged` 事件。状态变更时同步广播，所有订阅者在同一帧内响应。

  

**收益**：

- `TransitionTo()` 加一个断点 = 追踪一切状态变更

- 新增状态只需在 switch 中加 case，不影响未订阅的系统

- 去除 Update 中的守卫轮询，减少每帧开销

  

---

  

### ADR-002：为什么主状态和导览模式分开？

  

**方案 A**：全部合并到一个枚举

  

```

IdleNoGuide, IdleAutoGuide, IdleManualGuide,

TalkingNoGuide, TalkingAutoGuide, TalkingManualGuide,

GuidingAuto, GuidingManual, Moving...

```

  

→ 组合爆炸，12+ 个枚举值

  

**方案 B（采用）**：正交分离

  

```

AppState:  Idle | Talking | Guiding | Moving     (4 值)

GuideMode: None | Auto | Manual                   (3 值)

```

  

→ 两个独立维度，各自变更互不影响。UI 决策时组合查询即可。

  

---

  

### ADR-003：为什么不用 HideReason Flags 方案？

  

上一轮讨论中提出过引用计数 + Flags 的方案：

  

```csharp

[Flags] enum HideReason { InQA, GuideActive, Moving, ... }

RequestHide(reason) / ReleaseHide(reason)

```

  

**该方案的局限**：

- 只解决了 LanguageChange 显隐冲突，没有解决 mainState 散落和 13 个文件轮询的问题

- 如果将来有更多 UI 元素需要类似的显隐逻辑，每个元素都需要一套 HideReason

- 本质上是"给症状打补丁"，而不是"治疗病因"

  

**AppStateManager 方案的优势**：

- 从根源解决问题：一个权威状态 → 所有 UI 元素的显隐都在一个 switch 里决策

- 不仅解决 LanguageChange，同时解决 mainState 散落问题

- 未来新增任何 UI 控制需求，只需在 OnAppStateChanged 的 switch 里加一行

  

---

  

## 六、风险与注意事项

  

| 风险 | 缓解措施 |

|------|----------|

| 阶段一与旧代码并存期间可能出现双重控制 | 阶段一中旧方法仍保留但不主动调用；OnAppStateChanged 会覆盖旧方法的效果 |

| 事件订阅忘记取消导致内存泄漏 | OnDestroy 中统一取消订阅；代码审查时检查配对 |

| NetWorkGuide 状态变更未正确映射 | 阶段一先保留 Update 守卫作为兜底，阶段三再移除 |

| 多个 TransitionTo 在同一帧内连续调用 | TransitionTo 有相同状态去重；必要时可加入状态转换合法性检查 |