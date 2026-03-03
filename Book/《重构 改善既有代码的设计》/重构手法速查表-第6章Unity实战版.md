# 重构手法速查表 - 第6章(6.1-6.6) Unity实战版

> 来源：《重构：改善既有代码的设计》第6章
> 整理时间：2026-02-24
> 适用场景：Unity C# 游戏开发

---

## 📚 本章概览

第6章介绍了6个基础重构手法，是日常开发中最常用的技巧：

| 重构手法 | 核心作用 | 使用频率 |
|---------|---------|---------|
| 6.1 提炼函数 | 把大段代码拆成小函数 | ⭐⭐⭐⭐⭐ |
| 6.2 内联函数 | 合并过度拆分的函数 | ⭐⭐⭐ |
| 6.3 提炼变量 | 复杂表达式拆成有意义的变量 | ⭐⭐⭐⭐⭐ |
| 6.4 内联变量 | 删除多余的变量 | ⭐⭐⭐ |
| 6.5 改变函数声明 | 改函数名、调整参数 | ⭐⭐⭐⭐ |
| 6.6 封装变量 | 通过函数控制数据访问 | ⭐⭐⭐⭐⭐ |

---

## 🔧 6.1 提炼函数 (Extract Function)

### 一句话概括
**把一大段代码拆成多个小函数，让每个函数只做一件事。**

### 为什么需要？
- 100行的`Update()`方法难以理解
- 函数名就是注释，一眼看懂逻辑
- 小函数容易测试、复用、修改

### Unity实战对比

#### ❌ 重构前（混乱）
```csharp
void Update()
{
    // 检查输入
    float h = Input.GetAxis("Horizontal");
    float v = Input.GetAxis("Vertical");
    transform.position += new Vector3(h, 0, v) * 5f * Time.deltaTime;

    // 检查攻击
    if (Input.GetKeyDown(KeyCode.Space))
    {
        int baseDamage = 10;
        int criticalHit = Random.Range(0, 100) > 80 ? 2 : 1;
        int finalDamage = baseDamage * criticalHit;
        Debug.Log("造成伤害: " + finalDamage);
        GameObject.Find("DamageText").GetComponent<Text>().text = finalDamage.ToString();
    }

    // 检查生命值
    int currentHealth = GetComponent<Health>().CurrentHealth;
    if (currentHealth < 20)
    {
        Debug.LogWarning("血量过低!");
        GameObject.Find("HealthBar").GetComponent<Image>().color = Color.red;
    }
}
```

#### ✅ 重构后（清晰）
```csharp
void Update()
{
    HandleMovement();
    HandleAttack();
    CheckHealthStatus();
}

void HandleMovement()
{
    float h = Input.GetAxis("Horizontal");
    float v = Input.GetAxis("Vertical");
    transform.position += new Vector3(h, 0, v) * 5f * Time.deltaTime;
}

void HandleAttack()
{
    if (Input.GetKeyDown(KeyCode.Space))
    {
        int damage = CalculateDamage();
        ApplyDamage(damage);
        UpdateDamageUI(damage);
    }
}

int CalculateDamage()
{
    int baseDamage = 10;
    int critMultiplier = IsCriticalHit() ? 2 : 1;
    return baseDamage * critMultiplier;
}

bool IsCriticalHit() => Random.Range(0, 100) > 80;

void CheckHealthStatus()
{
    int currentHealth = GetComponent<Health>().CurrentHealth;
    if (currentHealth < 20)
    {
        ShowLowHealthWarning();
    }
}
```

### 重构步骤
1. 创建新函数，用意图命名（而非实现方式）
2. 将代码复制到新函数
3. 检查变量作用域
4. 替换原代码为函数调用
5. 测试
6. 查找其他相似代码重复处理

### 何时使用？
- ✅ 一个方法做了很多事
- ✅ 相同代码出现多次
- ✅ 需要添加注释才能理解的代码段
- ✅ 方法超过15行

---

## 🔧 6.2 内联函数 (Inline Function)

### 一句话概括
**把过度拆分的小函数合并回去，消除没必要的跳转。**

### 为什么需要？
- 函数名不如代码本身清晰
- 只有一行的包装函数
- 过度拆分反而让代码难懂

### Unity实战对比

#### ❌ 重构前（过度拆分）
```csharp
void Update()
{
    if (ShouldChasePlayer())
    {
        ChasePlayer();
    }
}

bool ShouldChasePlayer() => PlayerInRange();
bool PlayerInRange() => Vector3.Distance(transform.position, player.position) < 10f;
void ChasePlayer() => MoveTowards(player.position);
void MoveTowards(Vector3 target) => transform.position = Vector3.MoveTowards(transform.position, target, speed * Time.deltaTime);
```

#### ✅ 重构后（合理简化）
```csharp
void Update()
{
    float distanceToPlayer = Vector3.Distance(transform.position, player.position);
    if (distanceToPlayer < 10f)
    {
        transform.position = Vector3.MoveTowards(
            transform.position,
            player.position,
            speed * Time.deltaTime
        );
    }
}
```

### 重构步骤
1. 检查函数不具多态性
2. 找出所有调用点
3. 将函数调用替换为函数体
4. 每次替换后测试
5. 删除函数定义

### 何时使用？
- ✅ 函数体比函数名还清晰
- ✅ 只有一行代码且只调用一次
- ✅ 函数名很难起（说明可能不该拆分）

---

## 🔧 6.3 提炼变量 (Extract Variable)

### 一句话概括
**把复杂的表达式拆成多个有意义的变量，让代码像说人话。**

### 为什么需要？
- 复杂表达式难以理解
- 变量名就是注释
- 方便调试中间值

### Unity实战对比

#### ❌ 重构前（天书般的表达式）
```csharp
public float CalculatePrice(int quantity, float itemPrice)
{
    return quantity * itemPrice -
           Mathf.Max(0, quantity - 500) * itemPrice * 0.05f +
           Mathf.Min(quantity * itemPrice * 0.1f, 100);
}
```

#### ✅ 重构后（人话版代码）
```csharp
public float CalculatePrice(int quantity, float itemPrice)
{
    // 基础价格
    float basePrice = quantity * itemPrice;

    // 批量折扣（超过500个打95折）
    float quantityDiscount = Mathf.Max(0, quantity - 500) * itemPrice * 0.05f;

    // 运费（10%，最多100元）
    float shipping = Mathf.Min(basePrice * 0.1f, 100);

    // 最终价格
    return basePrice - quantityDiscount + shipping;
}
```

#### 在类中提炼为属性
```csharp
public class Enemy : MonoBehaviour
{
    public int maxHealth = 100;
    public int currentHealth = 80;
    public int armor = 20;

    // 提炼为属性，整个类都能用！
    public float HealthPercent => (float)currentHealth / maxHealth;
    public float EffectiveHealth => currentHealth + armor * 2;
    public bool IsInDanger => HealthPercent < 0.3f;

    void UpdateHealthBar()
    {
        healthBar.fillAmount = HealthPercent;  // 清晰！
    }

    void CheckDangerState()
    {
        isDangerous = IsInDanger;  // 更清晰！
    }
}
```

### 重构步骤
1. 确认表达式无副作用
2. 声明不可修改的变量
3. 用变量替换表达式
4. 测试
5. 如有多处出现，重复替换

### 何时使用？
- ✅ 表达式太长（超过一行）
- ✅ 有魔法数字
- ✅ 多处重复计算
- ✅ 调试时需要查看中间值

---

## 🔧 6.4 内联变量 (Inline Variable)

### 一句话概括
**把没必要的变量删掉，直接用表达式。**

### 为什么需要？
- 变量名不如表达式清晰
- 为了变量而变量
- 多余的跳转让代码啰嗦

### Unity实战对比

#### ❌ 重构前（啰嗦的变量）
```csharp
void Update()
{
    bool spacePressed = Input.GetKeyDown(KeyCode.Space);
    if (spacePressed)
    {
        Jump();
    }

    bool isActive = gameObject.activeSelf;
    if (isActive)
    {
        DoSomething();
    }

    float speed = 5f;
    transform.position += Vector3.forward * speed * Time.deltaTime;
}
```

#### ✅ 重构后（简洁版）
```csharp
void Update()
{
    if (Input.GetKeyDown(KeyCode.Space))
    {
        Jump();
    }

    if (gameObject.activeSelf)
    {
        DoSomething();
    }

    transform.position += Vector3.forward * 5f * Time.deltaTime;
}
```

### 重构步骤
1. 检查表达式无副作用
2. 声明为不可修改（测试）
3. 找第一处使用点，替换为表达式
4. 测试，重复直到全部替换
5. 删除变量声明
6. 最终测试

### 何时使用？
- ✅ 变量名等于表达式含义
- ✅ 只用一次的简单值
- ✅ 表达式本身就是文档

---

## 🔧 6.5 改变函数声明 (Change Function Declaration)

### 一句话概括
**给函数改个好名字、调整参数，让接口更清晰、耦合更少。**

### 为什么需要？
- 好的函数名一眼看懂用途
- 合理的参数减少耦合
- 提高函数复用性

### Unity实战场景

#### 场景1：改函数名

```csharp
// ❌ 重构前
public float circum(float radius) { }
public void doIt(GameObject obj) { }

// ✅ 重构后
public float CalculateCircumference(float radius) { }
public void DisableGameObject(GameObject obj) { }
```

#### 场景2：添加参数

```csharp
// ❌ 重构前（功能固定）
public void ReserveBook(Book book, Customer customer)
{
    normalQueue.Add(new Reservation(book, customer));
}

// ✅ 重构后（增加灵活性）
public void ReserveBook(Book book, Customer customer, bool isPriority = false)
{
    if (isPriority)
        priorityQueue.Add(new Reservation(book, customer));
    else
        normalQueue.Add(new Reservation(book, customer));
}
```

#### 场景3：参数改为属性（降低耦合）⭐⭐⭐

```csharp
// ❌ 重构前（过度耦合）
public bool IsInNewEngland(Customer customer)
{
    string[] states = { "MA", "CT", "ME", "VT", "NH", "RI" };
    return states.Contains(customer.address.state);
}

// 使用时必须有整个Customer对象
if (IsInNewEngland(customer)) { }

// ✅ 重构后（降低耦合）
public bool IsInNewEngland(string stateCode)
{
    string[] states = { "MA", "CT", "ME", "VT", "NH", "RI" };
    return states.Contains(stateCode);
}

// 使用时更灵活
if (IsInNewEngland(customer.address.state)) { }  // 从对象取
if (IsInNewEngland("MA")) { }  // 直接用字符串
```

### 两种做法

**简单做法**（调用者少）：
1. 直接修改函数声明
2. 修改所有调用处
3. 测试

**迁移式做法**（调用者多/多态函数）：
1. 创建新函数（新签名）
2. 老函数标记`[Obsolete]`并调用新函数
3. 逐步迁移调用处
4. 删除老函数

### 何时使用？
- ✅ 函数名看不懂
- ✅ 需要增加灵活性
- ✅ 参数耦合太强

---

## 🔧 6.6 封装变量 (Encapsulate Variable)

### 一句话概括
**不要直接访问数据，通过函数来读写，可以加验证、日志、监控。**

### 为什么需要？
- 直接访问无法控制
- 无法验证数据合法性
- 无法监听变化触发事件
- 调试时找不到谁改的

### Unity实战场景

#### 场景1：玩家血量

```csharp
// ❌ 重构前（随便改，出bug找不到原因）
public class Player : MonoBehaviour
{
    public int health = 100;
}

void SomeMethod()
{
    player.health = -50;  // Bug! 谁改的？不知道！
    player.health = 999;  // 作弊！无法检测！
}

// ✅ 重构后（完全掌控）
public class Player : MonoBehaviour
{
    [SerializeField] private int health = 100;
    [SerializeField] private int maxHealth = 100;

    // 只读属性
    public int Health => health;
    public int MaxHealth => maxHealth;

    // 修改血量的函数（加验证、日志、事件）
    public void SetHealth(int newHealth)
    {
        int oldHealth = health;
        health = Mathf.Clamp(newHealth, 0, maxHealth);  // 验证

        Debug.Log($"血量变化: {oldHealth} -> {health}");  // 日志

        if (health <= 0)
            OnDeath();  // 事件
        else if (health < maxHealth * 0.3f)
            OnLowHealth();  // 事件

        UpdateHealthBar();  // 更新UI
    }

    // 更方便的接口
    public void TakeDamage(int damage) => SetHealth(health - damage);
    public void Heal(int amount) => SetHealth(health + amount);
}
```

#### 场景2：深层封装（引用类型）

```csharp
// ❌ 重构前（列表可被外部修改）
public class GameConfig
{
    public static List<string> levelNames = new List<string> { "Level1", "Level2" };
}

void BadCode()
{
    GameConfig.levelNames.Clear();  // 糟糕！所有关卡都没了！
}

// ✅ 重构后（深层封装）
public class GameConfig
{
    private static List<string> levelNames = new List<string> { "Level1", "Level2" };

    // 返回副本，防止外部修改原数据！
    public static List<string> LevelNames => new List<string>(levelNames);

    // 或者用只读接口
    public static IReadOnlyList<string> LevelNamesReadOnly => levelNames.AsReadOnly();

    // 修改配置的函数（加验证）
    public static void AddLevel(string levelName)
    {
        if (string.IsNullOrEmpty(levelName))
        {
            Debug.LogError("Level name cannot be empty");
            return;
        }
        if (!levelNames.Contains(levelName))
        {
            levelNames.Add(levelName);
            Debug.Log($"Added level: {levelName}");
        }
    }
}

// 使用时
var levels = GameConfig.LevelNames;
levels.Clear();  // 不会影响原数据！
```

### 封装的3个层次

```csharp
// 层次1：基础封装（最常用）
private int health;
public int Health => health;
public void SetHealth(int value) { health = value; }

// 层次2：带验证封装
private int health;
public int Health => health;
public void SetHealth(int value)
{
    health = Mathf.Clamp(value, 0, maxHealth);
    OnHealthChanged?.Invoke();
}

// 层次3：深层封装（引用类型）
private List<Item> inventory = new List<Item>();
public IReadOnlyList<Item> Inventory => inventory.AsReadOnly();
public void AddItem(Item item) { inventory.Add(item); }
```

### 重构步骤
1. 创建读写函数
2. 静态检查
3. 逐一替换直接引用
4. 限制变量可见性
5. 测试

### 何时使用？
- ✅ 作用域超出单个函数的可变数据
- ✅ 需要验证数据
- ✅ 需要监听变化
- ✅ 引用类型需要防止外部修改

---

## 🎯 快速决策表

| 场景 | 使用手法 | Unity例子 |
|------|---------|-----------|
| 一个方法做了很多事 | **提炼函数** | `Update()`里处理输入、移动、战斗、UI |
| 函数名比代码还复杂 | **内联函数** | `IsTrue() { return true; }` |
| 表达式太长看不懂 | **提炼变量** | `a*b-c*d+e/f` → `basePrice - discount + tax` |
| 变量名没比代码清晰 | **内联变量** | `bool pressed = Input.GetKey(A)` → 直接用 |
| 函数名看不懂 | **改函数名** | `circum()` → `CalculateCircumference()` |
| 参数耦合太强 | **调整参数** | `Foo(Customer c)` → `Foo(string state)` |
| 数据被随意修改 | **封装变量** | `public int health` → `private` + 属性 |
| 需要验证数据 | **封装变量** | 限制血量范围 0-maxHealth |

---

## 💡 Unity最佳实践

### 1. 函数长度建议
```csharp
// ✅ 理想的函数长度：5-15行
void HandleMovement()  // 8行，完美
{
    float horizontal = Input.GetAxis("Horizontal");
    float vertical = Input.GetAxis("Vertical");
    Vector3 direction = new Vector3(horizontal, 0, vertical);
    direction.Normalize();
    transform.position += direction * speed * Time.deltaTime;
}
```

### 2. 变量封装模板
```csharp
public class Enemy : MonoBehaviour
{
    // 私有字段
    [SerializeField] private int health = 100;

    // 只读属性
    public int Health => health;

    // 修改方法（加验证）
    public void SetHealth(int value)
    {
        health = Mathf.Clamp(value, 0, maxHealth);
        OnHealthChanged?.Invoke();
    }
}
```

### 3. 参数设计原则
```csharp
// ❌ 糟糕：参数太多
void SpawnEnemy(GameObject prefab, Vector3 pos, Quaternion rot, int level, bool isBoss, float scale);

// ✅ 优秀：用配置对象
void SpawnEnemy(EnemySpawnConfig config);

public struct EnemySpawnConfig
{
    public GameObject Prefab;
    public Vector3 Position;
    public int Level;
    public bool IsBoss;
}
```

---

## 📊 重构流程图

```
遇到代码问题
    ↓
判断问题类型
    ↓
┌─────────────────────────────────┐
│  函数相关                        │
│  - 太长 → 提炼函数               │
│  - 太短 → 内联函数               │
│  - 名字差 → 改函数声明           │
│  - 参数多 → 调整参数             │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│  变量相关                        │
│  - 表达式长 → 提炼变量           │
│  - 变量多余 → 内联变量           │
│  - 随便改 → 封装变量             │
└─────────────────────────────────┘
    ↓
重构 → 测试 → 完成
```

---

## 🎓 核心原则

### 提炼 vs 内联
- **提炼**：为了理解和复用
- **内联**：为了简化和直观
- **平衡点**：函数名是否比代码本身更清晰

### 封装的价值
> "可变数据的作用域越大，封装的价值就越大。"

### 命名的重要性
> "好的名字让我们一眼看出函数的用途，而糟糕的名字则持续招致麻烦。"

---

## 📝 实战检查清单

### 代码审查时问自己：

**函数方面：**
- [ ] 这个函数是否做了多件事？（→ 提炼函数）
- [ ] 这个函数名是否清晰表达意图？（→ 改函数名）
- [ ] 参数是否只依赖必要的数据？（→ 调整参数）
- [ ] 这个函数是否只是简单包装？（→ 内联函数）

**变量方面：**
- [ ] 这个表达式是否一眼能看懂？（→ 提炼变量）
- [ ] 这个变量是否比表达式更清晰？（→ 内联变量）
- [ ] 这个数据是否可以被随意修改？（→ 封装变量）
- [ ] 引用类型是否可能被外部修改？（→ 深层封装）

---

## 🔗 相关资源

- 原书链接：[《重构：改善既有代码的设计》第6章](https://gausszhou.github.io/refactoring2-zh/ch6.html)
- Unity官方代码规范：[Unity C# Style Guide](https://unity.com/how-to/naming-and-code-style-tips-c-scripting-unity)

---

## 📌 记住这句话

> **"代码是写给人看的，顺便让机器执行。"**
>
> 重构的目的不是炫技，而是让代码更容易理解、修改和维护。
> 好的重构是在**过度简单**和**过度复杂**之间找到平衡。

---

*整理自《重构：改善既有代码的设计》第6章，结合Unity C#游戏开发实践*