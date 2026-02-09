![[Pasted image 20251224154125.png]]

## 核心原理解析：AActor 空间表达与组件继承

---

## 1. Actor 空间定位：RootComponent 代理机制

- **Actor 本质**：**无坐标容器**。`AActor` 类不直接存储 `Location`、`Rotation` 或 `Scale` 数据。
    
- **RootComponent (根组件)**：Actor 在 3D 空间中的**唯一坐标代表**。
    
- **数据流向**：
    
    - 物理数据存储于 `USceneComponent` 的 **Transform** 字段。
        
    - 调用 `GetActorLocation()` 时，Actor 内部通过**指针重定向**，实时查询 `RootComponent` 的空间状态。
        

---

## 2. 组件能力演进：继承与扩展

- **USceneComponent (基类)**：
    
    - **核心能力**：提供空间存在感（**Transform**）。
        
    - **逻辑定位**：所有需要挂载层级、拥有坐标的组件的共同祖先。
        
- **UStaticMeshComponent (子类)**：
    
    - **能力继承**：自动获得父类的 **Transform** 属性，支持移动与旋转。
        
    - **能力增强**：新增 **Mesh (模型数据)** 引用。
        
- **逻辑公式**：
    
    - `UStaticMeshComponent` = `USceneComponent` (空间坐标) + `UStaticMesh` (视觉模型)。
        

---

**想进一步了解这些组件在内存中的物理排布（Pointer Chasing）是如何影响 CPU 性能的吗？**