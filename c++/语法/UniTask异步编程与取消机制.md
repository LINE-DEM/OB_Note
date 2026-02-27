---
creation_date: 2026-02-24 15:00
type: #Type/Concept
status: #Status/Refactoring
tags: [UniTask, 异步编程, CancellationToken, Unity, C#]
aliases: [UniTask取消, 异步任务取消, Unity异步]
tech_stack: #Tech/Unity
complexity: ⭐⭐⭐⭐
related_modules: []
---

# UniTask 异步编程与取消机制

## 核心摘要

UniTask 是 Unity 专用的异步编程框架,通过 `async/await` 让异步操作像写同步代码一样简单,同时用 `CancellationToken` 控制任务的生命周期(随时取消运行中的任务)。

---

## 详细分析

### 底层原理 / 核心规律

**异步编程的核心规律**:
```
async 标记方法 → 方法内可以用 await 暂停 → 暂停时不阻塞主线程 → 完成后自动恢复执行
```

**取消机制的核心规律**:
```
CancellationTokenSource 发出取消信号 → CancellationToken 检测到信号 → 抛出异常中断任务
```

---

### 具体用法(按难度从低到高排列)

#### 用法 1: 最基础的 async/await 结构

**看代码**
```csharp
private async UniTask CheckIsOverConversation(CancellationToken cancellationToken)
//      ↑↑↑↑↑  ↑↑↑↑↑↑↑↑                      ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
//      关键词  返回类型                        取消令牌
{
    await UniTask.Yield();  // 暂停一帧,让出 CPU
    //    ↑↑↑↑↑
    //    等待完成后自动继续
}
```

**拆解**
- `async`: 标记这是一个异步方法
- `UniTask`: Unity 优化的异步返回类型(比 C# 原生的 `Task` 性能更好,零 GC)
- `await`: 遇到耗时操作时暂停当前方法,但**不阻塞主线程**
- `UniTask.Yield()`: 暂停一帧(类似 `yield return null`)

**对比**
```csharp
// 不用异步:会卡死主线程
while (condition) {
    // 一直循环占用 CPU
}

// 用异步:每帧检测一次,不卡顿
while (condition) {
    await UniTask.Yield();  // 每帧让出控制权
}
```

**为什么用**: 游戏中常见的"等待某个条件满足"的场景,用异步可以避免卡主线程。

---

#### 用法 2: CancellationToken 的取消检测

**看代码**
```csharp
while (AudioReceiver.AudioStringQueue.Count>0 || AudioMgr.Instance.isAnwsering)
{
    cancellationToken.ThrowIfCancellationRequested();
    //                ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
    //                检查是否被取消,如果是则抛异常

    await UniTask.Yield();
}
```

**拆解**
- `ThrowIfCancellationRequested()`: 检查外部是否发出了取消信号
- 如果被取消 → 抛出 `OperationCanceledException` 异常 → 中断整个方法

**为什么用**: 防止任务无限运行。例如用户关闭了对话界面,但检测任务还在后台跑,就需要立即停止。

---

#### 用法 3: 捕获取消异常

**看代码**
```csharp
try
{
    while (condition)
    {
        cancellationToken.ThrowIfCancellationRequested();
        await UniTask.Yield();
    }
}
catch (OperationCanceledException)
//     ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
//     专门捕获"取消"异常
{
    Debug.Log("任务已取消");  // 正常退出,不是错误
}
```

**拆解**
- `OperationCanceledException`: `ThrowIfCancellationRequested()` 抛出的异常类型
- 捕获后**静默处理**,因为取消不是错误,是预期行为

**为什么用**: 区分"任务被取消"和"任务出错"。取消是正常流程,出错才需要报警。

---

#### 用法 4: CancellationTokenSource 的创建和使用

**看代码**
```csharp
// 第1步: 创建取消源
_checkConversationCts = new CancellationTokenSource();
//                          ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
//                          取消信号的发射器

// 第2步: 启动任务,传入 Token
CheckIsOverConversation(_checkConversationCts.Token).Forget();
//                                            ↑↑↑↑↑
//                                            取消信号的接收器

// 第3步: 需要取消时,调用 Cancel()
_checkConversationCts.Cancel();
//                    ↑↑↑↑↑↑
//                    发出取消信号

// 第4步: 释放资源
_checkConversationCts.Dispose();
```

**拆解**
- `CancellationTokenSource`: 取消信号的**发送端**
  - `.Token` 属性: 取消信号的**接收端**(传给异步方法)
  - `.Cancel()` 方法: 发出取消信号
  - `.Dispose()` 方法: 释放资源

**生命周期流程**:
```
创建 Source → 传 Token 给任务 → 任务中检测 Token → 调用 Cancel() → 任务抛异常退出 → Dispose 清理
```

---

#### 用法 5: Forget() 的作用

**看代码**
```csharp
CheckIsOverConversation(_checkConversationCts.Token).Forget();
//                                                    ↑↑↑↑↑↑
//                                                    不等待任务完成
```

**拆解**
- `Forget()`: 告诉编译器"我不关心这个任务的返回值,让它在后台跑"
- 不加 `Forget()` 会有编译器警告: "你启动了一个异步任务但没有 await 它"

**对比**
```csharp
// 方式1: await 等待完成(会阻塞当前方法)
await CheckIsOverConversation(token);

// 方式2: Forget() 后台运行(不阻塞,立即继续)
CheckIsOverConversation(token).Forget();
```

**为什么用**: 打字机效果完成后,启动一个"持续监测"的任务,不需要等它完成,所以用 `Forget()`。

---

#### 用法 6: 完整的取消流程

**看代码**
```csharp
// 场景: 防止重复启动任务

// 步骤1: 取消旧任务
if (_checkConversationCts != null)
{
    _checkConversationCts.Cancel();   // 发出取消信号
    _checkConversationCts.Dispose();  // 释放资源
    _checkConversationCts = null;
}

// 步骤2: 启动新任务
_checkConversationCts = new CancellationTokenSource();
CheckIsOverConversation(_checkConversationCts.Token).Forget();
```

**拆解流程**:
```
旧任务还在跑? → 发 Cancel() 中断它 → Dispose 释放 → 创建新 Source → 启动新任务
```

**为什么用**: 常见场景是"用户快速点击按钮",需要停掉上一次的任务再启动新任务。

---

### 快速读法

遇到这种异步代码时,按此步骤理解:

1. **找 `async` 标记** → 这是异步方法
2. **找 `await` 点** → 这里会暂停,但不卡主线程
3. **找 `CancellationToken` 参数** → 这个任务可以被外部取消
4. **找 `ThrowIfCancellationRequested()`** → 这是取消检测点
5. **找 `try-catch OperationCanceledException`** → 这是取消后的退出逻辑
6. **找 `.Forget()`** → 这个任务是后台运行,不等待结果

---

### 常见模式总结

| 模式 | 代码特征 | 典型场景 |
|------|---------|---------|
| **持续检测** | `while(condition) { await Yield(); }` | 等待某个条件满足 |
| **可取消循环** | `while(condition) { token.ThrowIf...; await Yield(); }` | 长时间循环需要中断能力 |
| **后台任务** | `SomeAsyncMethod().Forget()` | 不关心返回值的异步操作 |
| **任务替换** | `Cancel() → Dispose() → new Source()` | 防止重复启动 |

---

## 关联知识

- [[协程与异步的区别]] (如果有)
- [[Unity生命周期]] (理解为什么要 Yield)
- [[C#异步编程]] (UniTask 是 Task 的优化版)

---

## 实战案例: 代码完整解读

```csharp
private async UniTask CheckIsOverConversation(CancellationToken cancellationToken)
{
    try
    {
        // 【核心逻辑】持续检测三个条件
        while (AudioReceiver.AudioStringQueue.Count>0 ||  // 音频队列非空
               AudioMgr.Instance.isAnwsering ||           // 正在回答
               !WSProtocolUtils.isTTSDone)                // TTS未完成
        {
            // 【取消检测】外部调用了 Cancel() 就立即退出
            cancellationToken.ThrowIfCancellationRequested();

            // 【暂停一帧】让出 CPU,下一帧再检测
            await UniTask.Yield();
        }

        // 【所有条件满足】执行后续逻辑
        if (AppStateManager.Instance.Current == AppStateManager.AppState.Talking)
        {
            MainUIForm.Instance?.SendMessage("KickAutoExitTimerFromTypewriter", "");
            UnityHandler.Instance.PlayAnimOld("YT_dj");  // 播放待机动画
            DestroyTypeWriter();
        }
    }
    catch (OperationCanceledException)
    {
        // 【正常退出】任务被取消不是错误
        Debug.Log("CheckIsOverConversation 任务已取消");
    }
}

// 【调用处】启动后台监测任务
if (_checkConversationCts != null)
{
    _checkConversationCts.Cancel();   // 停掉旧任务
    _checkConversationCts.Dispose();
}
_checkConversationCts = new CancellationTokenSource();
CheckIsOverConversation(_checkConversationCts.Token).Forget();  // 后台运行
```

**整体流程**:
```
启动任务 → 每帧检测条件 → 如果被 Cancel 就抛异常退出 → 条件全满足就执行后续逻辑 → 结束
```
