![Exported image](Exported%20image%2020260112170333-0.png)

其他API都是交给驱动做
      

**Memory Heaps**: 物理内存（VRAM DRAM）
 
**Memory Types**：只是描述内存 比如属性 行为方式  
1.**DEVICE_LOCAL** **是****VRAM** cpu只能通过传输命令来获取 适合存放高带宽资源  
2.**HOST_VISIBLE** **是****DRAM** cpu读取模型等大数据 然后DRAM用来暂存 然后通过Staging Buffer发送到VRAM上  
适合更新const buffer（ 通常指 ​​只读的Uniform Buffer 优先放于DEVICE_LOCAL堆（因高频访问）若数据需更新，通过HOST_VISIBLE暂存缓冲（Staging Buffer）复制到DEVICE_LOCAL​​）
 
3**.****DEVICE_LOCAL | HOST_VISIBLE** 现在只有AMD上有一点 256mb 适合constBuffer 但是别用满 因为显卡驱动会隐式使用这一块来存放描述信息 如果满了会分配到DRAM 所以使用的时候经过了PCIE 会非常慢
 
4.**HOST_VISIBLE | HOST_CACHED** 是 Vulkan 中一种​​特殊的内存类型组合​​，其底层实现依赖现代 CPU/GPU 架构的​​缓存一致性协议​​（如 PCIe 的缓存窥探机制）。
 ![Exported image](Exported%20image%2020260112170335-1.png)

![Exported image](Exported%20image%2020260112170342-2.png)