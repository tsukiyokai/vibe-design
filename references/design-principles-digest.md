# 设计原则速查

来源：`/Users/shanshan/repo/me/vibe-coding/references/design-code-principles/` 和 `/Users/shanshan/repo/me/vibe-coding/references/conventions/`
从SOLID、设计模式、C++语言特性中提炼与设计文档编写直接相关的决策框架。

---

## 1. 架构决策框架（SOLID视角）

设计文档中做架构决策时，用以下问题自检：

| 原则 | 自检问题                           | 违反信号                                |
| ---- | ---------------------------------- | --------------------------------------- |
| SRP  | 这个类/模块有几个改变的理由？      | 一个类同时处理协议解析和业务逻辑        |
| OCP  | 新增场景时需要修改现有代码吗？     | 每次加Layout都要改核心函数的if-else     |
| LSP  | 子类能在所有父类出现的地方替换吗？ | 子类override后改变了语义或抛出额外异常  |
| ISP  | 调用方是否被迫依赖它不需要的接口？ | 一个接口包含10个方法但大多数实现只用3个 |
| DIP  | 高层模块是否直接依赖低层实现？     | 算法层直接调用platform层的具体类        |

## 2. 设计模式选择表

设计文档"总体方案"章节选择架构方案时参考：

| 场景               | 推荐模式        | CANN中的实例                      |
| ------------------ | --------------- | --------------------------------- |
| 同族算子的统一结构 | Template Method | hccl算子的executor基类            |
| 多种通信算法可选   | Strategy        | selector选择ring/halving-doubling |
| 构建复杂配置对象   | Builder/Factory | 通信域初始化                      |
| 跨层事件通知       | Observer        | 状态变更通知                      |
| 资源生命周期管理   | RAII            | 内存、流、硬件事件                |
| 接口适配           | Adapter/Facade  | hccl封装hcomm接口                 |
| 行为状态切换       | State Machine   | 通信链路状态管理                  |
| 延迟初始化/代理    | Proxy           | 按需建链                          |

## 3. C++接口设计要点

设计文档"接口设计"章节需要决策的关键点：

- 多态实现方式：virtual（运行时）vs模板（编译时）vs CRTP（零开销多态）vs Type Erasure（值语义多态）
- 参数传递：输入用const ref，sink参数用value+move，输出用返回值（不用输出指针）
- 错误处理：返回Status/StatusOr vs异常vs错误码（CANN惯例：HcclResult错误码）
- 资源管理：RAII封装，禁止裸new/delete，shared_ptr仅在真正共享所有权时使用
- 值语义：Rule of Five，noexcept移动（影响vector扩容性能10x）

## 4. CANN特有架构模式

设计文档"现状分析"章节描述架构时的参考：

hcomm三层架构：
- algorithm层：集合通信算法实现（AllReduce/AllGather等）
- framework层：通信域管理、资源编排、API入口
- platform层：通信原语、传输通道、内存管理

hccl算子统一结构：
- ops/<algo>/ 下的_op.h（算子入口）
- executor/（执行器，继承基类）
- selector/（算法选择器）
- template/（通信模板）

依赖方向：hccl单向依赖hcomm（5类接口：通信域管理、集合操作、点对点、流同步、状态查询）

## 5. 并发与同步设计要点

设计文档涉及多线程或流水线时，需要决策的关键点：

thread_local使用规范:
- 仅用于线程私有的临时状态(如per-thread buffer)，不用于跨线程共享
- 必须提供初始值，线程入口处显式重置
- 析构时确保thread_local资源被释放(注意线程池场景线程可能不销毁)

atomic操作规范:
- 单一load或store：用memory_order_relaxed足够时不加强
- read-modify-write：使用compare_exchange_weak循环或fetch_add，禁止load+判断+store的非原子序列
- 发布-获取模式：producer用memory_order_release，consumer用memory_order_acquire

锁策略:
- 优先无锁设计(atomic/thread_local)，其次细粒度锁，最后粗粒度锁
- 禁止持锁调用外部模块接口(避免死锁)
- 多锁场景必须定义全局锁序(lock ordering)并在设计文档中标注

## 6. 硬件适配设计要点

设计文档涉及多设备类型或通信层级时，需要决策的关键点：

DevType分支规范:
- 新增硬件适配时，列出所有受影响的DevType分支(switch/if)
- 使用selector模式隔离硬件差异，避免在核心算法中散布设备判断
- default分支必须有明确行为(报错或fallback)，不留空

通信层级(L0/L1/L2)选择:
- L0(设备内)：DMA搬运，带宽最高延迟最低，约束为同设备内
- L1(节点内)：RDMA/PCIe/HCCS，支持节点内跨设备，需处理NUMA亲和性
- L2(节点间)：RDMA/TCP网络，延迟最高，需考虑带宽分配和拥塞控制
- 设计文档须明确每个数据搬运操作选用的通信层级及理由

设备常量引用规范:
- 硬件常量(如最大rank数、buffer对齐要求)必须通过平台抽象层查询，禁止硬编码
- 不同设备的常量差异须在设计文档的"约束与限制"章节列出

---

完整内容详见：
- 设计原则：`/Users/shanshan/repo/me/vibe-coding/references/conventions/design-and-coding-guide.md`
- 编码规范：`/Users/shanshan/repo/me/vibe-coding/references/conventions/cann-cpp-conventions.md`
- 代码架构：`/Users/shanshan/repo/me/vibe-coding/references/dig-repo-code/phase0-architecture-map.md`
