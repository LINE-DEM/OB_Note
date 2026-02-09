### 案例

#### A. 头文件 (.h)

注意：这里我们不需要 `UInputAction` 指针，因为我们使用硬编码的字符串名 称（"MoveForward"）来建立连接。 

C++

```
// LegacyCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "LegacyCharacter.generated.h"

UCLASS()
class MYGAME_API ALegacyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    ALegacyCharacter();

    // 核心绑定函数
    virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;

protected:
    // --- 轴映射回调 (每帧调用) ---
    // float Value: 来自硬件的输入值 (0.0 表示无输入)
    void MoveForward(float Value);
    void MoveRight(float Value);
    
    // 鼠标输入通常直接传给 Controller，但也可以在这里处理
    void TurnAtRate(float Rate);

    // --- 动作映射回调 (事件触发) ---
    void StartJump();
    void StopJump();
    void FireWeapon();
};
```

#### B. 源文件 (.cpp)

这里的核心是 `BindAxis` 和 `BindAction`。请注意它们的参数差异。

C++

```
// LegacyCharacter.cpp
#include "LegacyCharacter.h"
#include "GameFramework/PlayerController.h"
#include "Components/InputComponent.h" // 必须包含

void ALegacyCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    // 检查指针有效性 (这是 C++ 开发者的基本素养)
    check(PlayerInputComponent);

    // --- 1. 绑定轴 (Axis Bindings) ---
    // 语法：BindAxis("配置文件里的名字", 对象指针, &类名::函数名)
    // 警告：这些字符串必须与 Project Settings 里的完全一致，区分大小写！
    // 这是一个典型的 "Stringly Typed" 编程，也是它被废弃的原因之一（易出错）。
    
    PlayerInputComponent->BindAxis("MoveForward", this, &ALegacyCharacter::MoveForward);
    PlayerInputComponent->BindAxis("MoveRight", this, &ALegacyCharacter::MoveRight);

    // 鼠标直接绑定到 APawn 的基类函数上 (如果不需要特殊逻辑)
    // AddControllerYawInput 是 APawn 已经写好的修改 Controller 旋转的函数
    PlayerInputComponent->BindAxis("Turn", this, &APawn::AddControllerYawInput);
    PlayerInputComponent->BindAxis("LookUp", this, &APawn::AddControllerPitchInput);

    // --- 2. 绑定动作 (Action Bindings) ---
    // 语法：BindAction("名字", 触发时机, 对象指针, &类名::函数名)
    
    // IE_Pressed  = 按下瞬间 (Input.GetKeyDown)
    // IE_Released = 松开瞬间 (Input.GetKeyUp)
    // IE_Repeat   = 按住连发 (类似打字机的重复输入，很少用于游戏逻辑)
    
    PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &ALegacyCharacter::StartJump);
    PlayerInputComponent->BindAction("Jump", IE_Released, this, &ALegacyCharacter::StopJump);
    
    PlayerInputComponent->BindAction("Fire", IE_Pressed, this, &ALegacyCharacter::FireWeapon);
}

// 逻辑实现：标准的矩阵变换
void ALegacyCharacter::MoveForward(float Value)
{
    // 优化：浮点数比较，避免微小噪音导致的不必要计算
    // KINDA_SMALL_NUMBER 是 UE 定义的一个极小值 (1.e-4)
    if ((Controller != nullptr) && (Value != 0.0f))
    {
        // 1. 找出 "前方" 是哪里
        // 我们通常使用 Controller 的旋转，而不是 Character 自身的旋转
        // 这样可以实现 "独立视角" (Free Look) 的移动
        const FRotator Rotation = Controller->GetControlRotation();
        
        // 2. 只需要 Yaw (水平朝向)，忽略 Pitch (抬头/低头) 和 Roll
        // 如果你不忽略 Pitch，角色可能会飞向天空或钻入地下
        const FRotator YawRotation(0, Rotation.Yaw, 0);

        // 3. 获取前向单位向量 (Unit Vector)
        // EAxis::X 在 UE 坐标系中代表 Forward (红色轴)
        const FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);

        // 4. 施加移动
        // Value 会决定正向还是反向 (W=1.0, S=-1.0)
        AddMovementInput(Direction, Value);
    }
}

void ALegacyCharacter::MoveRight(float Value)
{
    if ((Controller != nullptr) && (Value != 0.0f))
    {
        const FRotator Rotation = Controller->GetControlRotation();
        const FRotator YawRotation(0, Rotation.Yaw, 0);

        // EAxis::Y 在 UE 坐标系中代表 Right (绿色轴)
        const FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);

        AddMovementInput(Direction, Value);
    }
}

void ALegacyCharacter::StartJump()
{
    bPressedJump = true; // ACharacter 内置变量，触发跳跃逻辑
}

void ALegacyCharacter::StopJump()
{
    bPressedJump = false;
}

void ALegacyCharacter::FireWeapon()
{
    // 具体的开火逻辑...
}
```

---

### 架构师视角的深度解析 (Controller)

**Pawn-Controller分离**设计，本质是为了支持**AI和玩家共享同一套Pawn逻辑**。



#### 核心定位

Controller在UE中是一个**无物理表示的逻辑实体**（没有Transform，不在场景中渲染），它的内存生命周期和Pawn是**解耦**的：

cpp

```cpp
// 架构关系（伪代码示意）
AController* Controller;  // 可以在多个Pawn之间切换
APawn* PossessedPawn;     // Controller当前控制的Pawn

// 经典控制流
Controller->Possess(PawnA);  // 占有
Controller->UnPossess();     // 释放
Controller->Possess(PawnB);  // 切换到另一个Pawn
```

#### 设计模式映射

从你的设计模式角度看，Controller是**策略模式(Strategy) + 状态模式(State)**的混合体：

- **PlayerController** = 人类玩家策略（接收输入）
- **AIController** = AI策略（接收感知系统输入，如EQS）
- 内部状态：Idle → Moving → Dead 等生命周期状态

#### 内存与执行层细节

当你调用 `Controller->GetPawn()` 时，这**不是**虚函数查表，而是直接的指针解引用（存储在`APawn* Pawn`成员中）。但当你调用 `Controller->OnPossess()` 时，**这才是**虚函数，因为PlayerController和AIController需要不同的占有逻辑。

从汇编角度，Possess过程大致是：



```asm
; 1. 检查旧Pawn并解绑
mov rax, [rcx+PawnOffset]   ; 读取Controller->Pawn指针
test rax, rax               ; 判空
jz skip_unpossess
call UnPossessInternal      ; 虚函数调用（查vptr）

; 2. 绑定新Pawn
skip_unpossess:
mov [rcx+PawnOffset], rdx   ; Controller->Pawn = NewPawn
mov [rdx+ControllerOffset], rcx  ; NewPawn->Controller = this
call OnPossess_Virtual      ; 触发蓝图事件钩子
```
