---
name: vibe-design
description: "用于生成软件设计文档。当用户提到 '设计文档'、'design doc'、'SD'、'SRS'、'需求分析'、'软件设计'、'架构设计'，或要求根据需求描述生成技术设计文档时触发。自动学习代码仓库中的相关代码和文档，基于模板生成 markdown 格式的设计文档（含 4+1 架构视图的 PlantUML/D2 图），并转换为 .docx 文件。"
---

# 软件设计文档生成Skill

## 概述

根据用户输入的需求描述，通过结构化探索协议深入理解代码仓库，经用户确认后基于选定模板生成设计文档，并转换为Word文件。

## 快速参考

| 阶段            | 操作                                         | 产出              |
| --------------- | -------------------------------------------- | ----------------- |
| 1. 结构化探索   | 三层并行探索：鸟瞰 → 深潜 → 横切             | 知识地图 + 符号表 |
| 2. 知识对齐     | Checkpoint 1：向用户呈现理解摘要，确认或纠正 | 确认的架构理解    |
| 3. 模板与大纲   | 选模板 + Checkpoint 2：章节深度分配          | 确认的写作计划    |
| 4. 生成Markdown | 生成文档 + 格式化 + 渲染图表 + 内容自检 + 定稿 | 定稿的`<name>.md` + PNG/SVG |
| 5. 图表渲染     | (第四步子流程)PlantUML→PNG + D2→SVG             | PNG/SVG图片                 |
| 6. 格式转换     | 生成 .docx + 产物校验                            | `<name>.docx`               |

### 输出约定

- 默认输出目录：目标仓库下的 `docs/design/`（不存在时自动创建）
- 用户可在第一步探索前指定其他路径，后续步骤统一使用
- 所有产物(md、png、svg、docx)输出到同一目录

---

## 第一步：结构化探索

收到需求描述后，先与用户确认探索范围（涉及哪些模块、关注什么问题），然后通过三层探索构建知识体系。每层使用Explore subagent执行，Layer 2按模块并行。

```
Layer 1: Bird's Eye (Explore subagent)
  directory structure -> build system -> module boundaries
  Output: module map

Layer 2: Deep Dive (Explore subagent x N, parallel by module)
  headers/interfaces -> key data structures -> core function signatures
  Output: interface inventory + symbol table

Layer 3: Cross-Cut (Explore subagent)
  inter-module call chains -> data flow -> error handling paths
  Output: interaction graph
```

### 1.1 Layer 1: 鸟瞰

启动一个Explore subagent，任务：
- 扫描顶层目录结构和构建系统（CMakeLists.txt、Makefile、BUILD等）
- 识别模块边界（哪些目录是独立模块，模块间的依赖关系）
- 读README、docs/ 目录建立领域概念
- 产出：模块地图（模块名 → 目录 → 职责 → 依赖）

### 1.2 Layer 2: 深潜

根据用户指定的核心模块，为每个模块启动一个Explore subagent并行执行：
- 读该模块的头文件(.h)，提取公开接口
- 识别关键数据结构(class、struct、enum)及其字段含义
- 提取核心函数签名（参数、返回值、语义）
- 追溯关键函数的实现逻辑（不是逐行读，而是理解控制流和数据流）
- 产出：接口清单 + 符号表（类名、函数名、数据结构，附file:line）

符号表格式：
```
| 符号 | 类型 | 文件 | 行号 | 说明 |
|------|------|------|------|------|
| ZeroCopyAddressMgr | class | zero_copy_address_mgr.h | 25 | 零拷贝地址管理 |
| GetLocalIpc2RemoteAddr | method | zero_copy_address_mgr.cc | 131 | IPC地址映射查找 |
```

### 1.3 Layer 3: 横切

启动一个Explore subagent，基于Layer 1/2的产出：
- 追踪模块间的调用链（A模块的哪个函数调用了B模块的哪个接口）
- 识别数据流（数据从哪里产生、经过哪些变换、最终到哪里）
- 识别错误处理路径（异常如何传播、在哪里被捕获）
- 产出：模块间交互关系（调用方 → 被调方，附file:line）

### 1.4 何时算"探索充分"

三层探索完成后，检查是否能回答：
1. 需求涉及哪些模块，各模块的职责是什么？
2. 核心数据结构是什么，字段含义是什么？
3. 现有的执行路径是什么？修改后会变成什么？
4. 模块间如何交互？依赖关系是什么？
5. 有哪些约束和限制？

如果有问题回答不了，针对性地补充探索（可以再派subagent），不要带着盲区开始写。

### 1.5 CANN领域探索增强(可选)

当目标仓库属于CANN生态(hccl/hcomm/ops-transformer/ops-nn等)时，加载以下参考资料辅助后续设计:
- references/cann-defect-patterns.md — 8大缺陷模式及防范规则
- references/design-principles-digest.md — 设计决策框架和CANN架构模式

#### 1.5.1 探索增强项

在通用三层探索之上，叠加CANN特化的查找目标：

Layer 1增强(鸟瞰阶段追加):
- 识别代际归属：涉及的代码属于V1/V2/legacy/next哪个代际? 查看目录名、命名空间、宏定义(如HCCL_V2)
- 识别硬件目标范围：涉及哪些DevType枚举值? 查看设备类型分支和条件编译宏
- 识别跨仓依赖：检查include路径和构建依赖，判断改动是否涉及另一个仓库(如hccl改动需要hcomm配合新增接口)。若涉及，将对端仓库自动纳入后续探索范围

Layer 2增强(深潜阶段追加):
- 识别分派链路径：追踪selector→executor→template的分派链，确认涉及哪些具体类
- 识别注册宏：查找REGISTER_EXEC_V2、REGISTER_OP等注册宏，确认算子/执行器的注册方式
- (跨仓场景)对端仓库深潜：对涉及的对端模块执行同等深度的接口提取和符号表构建

Layer 3增强(横切阶段追加):
- 追踪三层通信层级：确认涉及L0(设备内DMA)、L1(节点内RDMA/PCIe)、L2(节点间网络)中的哪些层级
- 追踪flag/barrier配对：识别流水线同步点，确认每个DataCopy/Compute的同步设计
- 追踪跨代调用：是否存在V1代码调用V2接口或反向调用的情况
- (跨仓场景)追踪跨仓调用链：从调用方仓库到被调方仓库的完整路径，明确接口边界(哪些函数是跨仓入口)

#### 1.5.2 充分性检查增强

探索阶段的充分性检查额外增加以下必答问题:
6. 方案是否触及已知缺陷模式? 哪些模式需要在设计中主动防范?
7. 改动涉及哪个代际(V1/V2)? 是否有跨代兼容要求?
8. 影响哪些硬件目标(DevType)? 分支覆盖是否完整?
9. 涉及哪些通信层级(L0/L1/L2)? 协议约束是什么?
10. (跨仓场景)改动是否涉及多个仓库? 各仓库的变更边界和提交顺序是什么?

---

## 第二步：知识对齐(Checkpoint 1)

探索完成后，必须向用户呈现知识摘要，确认理解正确后才能继续。

呈现内容：
- 模块职责：每个相关模块做什么
- 核心接口：关键的类、函数、数据结构
- 模块交互：谁调用谁、数据怎么流动
- 设计意图：为什么现有代码这样设计（如果能判断的话）
- 盲区标注：明确列出"我不确定"或"没找到"的部分
- 输出文件名：从需求标题派生文件名（如`非对称跨框场景TopoMatch适配`），向用户显式确认。后续所有步骤(md/png/svg/docx)统一使用此文件名前缀
- (CANN项目追加)代际归属与兼容性：涉及V1/V2哪个代际，是否有跨代约束
- (CANN项目追加)硬件目标范围：涉及哪些DevType，分支覆盖情况
- (CANN项目追加)分派链路径：涉及哪些selector→executor→template链路
- (CANN项目追加)跨仓接口边界：哪些改动在本仓、哪些需要对端仓库配合，接口契约是什么，建议的提交顺序

用户可能会：
- 确认 → 进入第三步
- 纠正 → 修正理解，必要时补充探索
- 补充 → 提供额外的背景信息或设计约束

不要跳过这一步。闷头写出来的文档大概率需要推倒重来。

---

## 第三步：模板选择与大纲规划(Checkpoint 2)

### 3.1 模板选择

| 条件               | 模板                                      |
| ------------------ | ----------------------------------------- |
| 用户指定了模板路径 | 使用用户指定的模板                        |
| CANN生态仓库(hccl/hcomm等) | [srs-sd.md](assets/templates/srs-sd.md)   |
| 用户要求SRS+SD格式 | [srs-sd.md](assets/templates/srs-sd.md)   |
| 默认 / 通用场景    | [generic.md](assets/templates/generic.md) |

### 3.2 大纲与深度分配

列出计划写的每个章节，标注深度等级，向用户确认：

```
示例：
  3.1 ZeroCopy地址管理    [详写] — 核心改动，需要完整的设计描述
  3.2 通信链路建立         [略写] — 未涉及变更，简述现状即可
  3.3 错误处理机制         [中等] — 有间接影响，需要说明兼容性
```

深度等级：
- 详写：完整的设计描述，含架构图、接口定义、数据流、设计决策
- 中等：描述现状和变更影响，含必要的图表
- 略写：一两段概述，不画图

用户确认后再开始正文。重点章节的篇幅至少是略写章节的3倍。

---

## 第四步：生成Markdown设计文档

严格按照选定模板的标题层级和章节结构生成markdown文件。

### 4.1 写作准则

禁止（反模式）：
- 禁止逐行翻译代码为中文描述 — 设计文档不是代码注释
- 禁止每个章节均匀用力 — 必须按Checkpoint 2的深度分配
- 禁止只写"是什么" — 每个设计点必须回答"为什么这样做"
- 禁止凭记忆写代码细节 — 不确定就回去Grep确认
- 禁止在正文中堆砌防御性警告 — 写作阶段关注设计本身，缺陷子模式仅用于第六步自检

要求（正模式）：
- 先结论再展开：这个模块解决什么问题 → 怎么解决的 → 为什么选这个方案
- 每个架构描述附带代码引用(file:line)，让读者能追溯
- 站在"向新人解释这个系统"的视角，而非"向编译器解释代码"
- 设计决策要说明取舍：选了A方案，B方案为什么不行
- 多画图：能用图表达的不用文字。拓扑关系、数据流向、调用链路、状态转换都应优先用图呈现，文字作为图的补充说明而非替代
- D2触发检查：写完每个章节后，扫描是否存在以下内容，若存在则必须生成对应的D2图而非纯文字/ASCII:
  1. rank/设备分布描述 → rank拓扑图
  2. 数据搬运或通信路径描述 → 数据流图
  3. 层级分组描述(L0/L1/L2, level 0/level 1) → 通信层级图
  4. 流水线阶段描述 → 流水线同步图
  ASCII代码块中的拓扑示意不能替代D2图——ASCII是草稿，D2是正式交付物
- 表格 vs 定义列表的选择：根据条目复杂度自动判断。每个条目仅1-2个属性(如术语→释义、枚举→含义)用表格；每个条目3个以上属性(如参数的类型+来源+说明+示例)用竖排定义列表(四级标题+子项展开)
- 对照cann-defect-patterns.md的"按设计文档章节的关联映射"表(仅8大模式，不含子模式)，在对应章节主动回答防范问题(如: 接口参数类型够宽吗? 流水线切换有barrier吗?)。子模式详解仅在第六步自检时使用
- 设计决策要引用design-principles-digest.md中的框架(如: 选择Strategy模式是因为需要运行时切换算法)

### 4.2 PlantUML硬约束

图表准确性是底线，遵循以下规则：

1. 图中出现的每个类名、函数名，必须存在于Layer 2构建的符号表中
2. 调用关系必须基于Layer 3确认的实际调用链，不凭印象画箭头
3. 如果符号表中没有，先用Grep确认代码中存在，再加入图表
4. 图表元素旁用PlantUML注释标注来源文件（不显示在渲染结果中）：
   ```plantuml
   class ZeroCopyAddressMgr {  ' zero_copy_address_mgr.h:25
       +GetLocalIpc2RemoteAddr()  ' zero_copy_address_mgr.cc:131
   }
   ```
5. 绘制完成后逐元素自检：对每个类名/函数名回查代码，发现不符立即修正

PlantUML编写规范和视图选择指南参见 [plantuml-guide.md](references/plantuml-guide.md)。

### 4.2b D2示意图规则

D2图不是可选装饰，而是特定内容的必选表达方式。当文档中出现拓扑布局、通信层级、数据流向、流水线阶段等内容时，必须用D2图表达，文字仅作为D2图的补充说明。

PlantUML适合UML图(类图/时序图/活动图/组件图)，D2适合非UML示意图(拓扑布局/数据流/架构总览)。选择依据：

| 需要表达的内容 | 选用工具 |
| -------------- | -------- |
| 类继承/接口关系 | PlantUML |
| 调用时序/状态转换 | PlantUML |
| rank/设备拓扑、空间布局 | D2 |
| 通信层级、数据搬运路径 | D2 |
| 流水线阶段与同步点 | D2 |

D2图编写规则：
1. 图中出现的设备名、rank标识、模块名必须与代码/配置一致
2. 使用容器(container)表达层级关系(pod→node→device→rank)
3. 使用grid布局表达空间排列(如rank矩阵)
4. 连接标签标注通信层级(L0/L1/L2)或数据量
5. 图表元素旁用D2注释标注来源：`# file:line`
6. 绘制完成后逐元素自检，与PlantUML同等要求
7. 不使用颜色(fill/stroke)区分元素 — 渲染时使用 `--sketch -t 0` 黑白手绘风格，用stroke-dash、形状、标签文字区分不同连线/分组
8. 表达N对M映射关系(如rank→slot、子组→层级)时，优先使用grid矩阵布局(行=一个维度，列=另一个维度，交叉点=映射结果)，而非逐条连线。连线图在节点超过8个时会因交叉导致可读性急剧下降
9. 宽高比控制：渲染后检查图片宽高比，超过3:1或1:3时必须调整布局。常见原因及修复：grid-rows:1导致横向铺开→增加grid-rows使元素折行；direction与容器嵌套方向一致导致单轴拉伸→调整direction或拆分容器。目标宽高比在4:3到3:4之间

D2编写规范和CANN场景示例参见 [d2-guide.md](references/d2-guide.md)。

### 4.3 其他写作要求

- 所有模板中定义的章节必须存在且非空
- 根据需求选用2-4个最相关的4+1架构视图
- 中文优先：文档内容、图中标签均使用中文
- 文件名必须使用Checkpoint 1中确认的文件名，禁止使用`design_doc`、`output`等通用名称。文件名从需求标题派生，如 `自定义算子支持aclGraph-ccu模式.md`

### 4.4 渲染

Markdown文件生成完成后(以及后续任何修改导致的刷新后)，提取并渲染图表(PlantUML→PNG，D2→SVG)，更新图片引用(详见第五步)。

### 4.5 内容自检与定稿(Checkpoint 3)

渲染完成后，执行内容层面的质量自检。此时用户可以看到渲染后的图表，审稿体验完整。

文档自检：

| 检查项         | 规则                                                  | 修复方式               |
| -------------- | ----------------------------------------------------- | ---------------------- |
| 必填章节完整性 | 模板定义的所有章节均必须存在且非空                    | 补充缺失章节内容       |
| 表格完整性     | 所有表格不得有空单元格，表头与数据行列数一致          | 填充空单元格或修正列数 |
| 代码块语言标注 | 所有代码块必须标注语言                                | 给裸代码块补上语言标注 |
| D2覆盖率       | 文档中不得存在未可视化的拓扑/数据流/层级ASCII描述；每个此类描述必须有对应的D2图 | 将ASCII描述转为D2代码块并渲染 |
| 图片去重       | 同一张图不得在多个章节重复引用；若两处语境不同则应分别绘制各自侧重的图 | 为不同章节画独立的图，调整视角和详略 |

与代码一致性校验：

| 检查项         | 方法                           | 修复方式             |
| -------------- | ------------------------------ | -------------------- |
| 文件路径有效   | 用Glob验证文档中引用的路径存在 | 修正为实际路径       |
| 函数签名一致   | Grep验证函数签名存在且一致     | 修正签名为代码实际值 |
| 枚举/宏值正确  | 对比文档引用值与代码定义       | 修正为代码实际值     |
| 结构体字段一致 | 对比文档中的字段列表与代码定义 | 增删字段使其一致     |

缺陷防范审查(CANN项目适用):

| 检查项             | 方法                                                              | 修复方式                 |
| ------------------ | ----------------------------------------------------------------- | ------------------------ |
| 整数溢出风险       | 检查接口设计中的数值参数是否使用int64_t/uint64_t                  | 修正类型或说明选择理由   |
| 分支覆盖           | 新增枚举/Layout时是否列出所有受影响的switch/if                    | 补充影响分析             |
| 构建注册           | 新算子是否明确CMake+config+yaml注册清单                           | 补充注册清单             |
| Host-Kernel一致性  | TilingData是否通过共享头文件? workspace计算是否一致?              | 修正设计或标注风险       |
| 流水线同步         | DataCopy流程是否设计了正确的barrier/flag配对?                     | 补充同步设计             |
| 变更规模           | 预估代码变更是否超3000行? 是否需要拆分?                          | 补充拆分方案             |

自检修改后重新渲染图表，然后向用户呈现最终md(含渲染图)，确认定稿后再进入第六步生成docx。

---

## 第五步：图表渲染

### 5.1 安装检查

```bash
# PlantUML
# macOS
brew install plantuml
# Linux
apt-get install -y plantuml
# 或手动下载 jar
curl -L -o /tmp/plantuml.jar https://github.com/plantuml/plantuml/releases/latest/download/plantuml.jar

# D2
# macOS
brew install d2
# Linux
curl -fsSL https://d2lang.com/install.sh | sh -s --
```

### 5.2 提取与渲染

1. 创建临时目录：`mkdir -p /tmp/designdoc_diagrams`
2. 从markdown中提取plantuml代码块为`.puml`文件，d2代码块为`.d2`文件
3. 渲染PlantUML：`plantuml -tpng -o /tmp/designdoc_diagrams/ /tmp/designdoc_diagrams/*.puml`
4. 渲染D2：对每个.d2文件执行 `d2 --sketch -t 0 input.d2 output.svg`（黑白手绘风格）
5. 宽高比验证：对每张渲染产物检查尺寸（`sips -g pixelWidth -g pixelHeight`或`identify`），宽高比超过3:1或1:3的图必须回到.d2源码调整布局后重新渲染
6. 更新markdown中的图片引用：plantuml代码块替换为`![标题](path.png)`，d2代码块替换为`![标题](path.svg)`

详细的PlantUML编写规范参见 [plantuml-guide.md](references/plantuml-guide.md)，D2编写规范参见 [d2-guide.md](references/d2-guide.md)。

---

## 第六步：格式美化与Word转换

### 6.1 生成 .docx

使用docx-js (`npm install -g docx`)按排版规范生成 .docx文件。完整的排版规范、样式定义和代码示例参见 [docx-formatting.md](references/docx-formatting.md)。

图片处理：docx不支持SVG嵌入，生成docx前需将D2产出的SVG转为PNG：
```bash
# 使用rsvg-convert(推荐，保真度高)
# macOS: brew install librsvg    Linux: apt-get install -y librsvg2-bin
rsvg-convert -o output.png input.svg

# 或使用d2直接输出PNG
d2 --theme 200 input.d2 output.png
```

核心要求：
- 不生成封面页，仅目录 + 正文
- 带编号的层级标题(H1 ~ H4)
- 统一的表格、代码块、图片、列表样式
- 页眉页脚（页眉右对齐项目名称，页脚居中页码）
- 标题层级必须严格匹配所选模板

### 6.2 产物校验

渲染和转换完成后，执行产物层面的检查：

| 检查项         | 规则                                                  | 修复方式                   |
| -------------- | ----------------------------------------------------- | -------------------------- |
| PlantUML可编译 | 所有 `.puml` 文件必须通过 `plantuml -syntax` 语法检查 | 修复语法错误后重新渲染     |
| D2可编译       | 所有 `.d2` 文件必须通过 `d2 fmt --check` 检查         | 修复语法错误后重新渲染     |
| 图片引用有效   | 所有 `![...](path)` 引用的图片文件必须存在            | 重新渲染缺失图片或修正路径 |

若产物校验修改了Markdown，重新渲染图表并重新生成.docx。

---

## 完整工作流

```
User input (requirement description)
    |
[1] Structured exploration (3 layers, parallel subagents)
    |  L1: bird's eye   -> module map
    |  L2: deep dive    -> interface inventory + symbol table
    |  L3: cross-cut    -> interaction graph
    |
[2] Checkpoint 1: present understanding summary -> user confirms
    |
[3] Select template + Checkpoint 2: outline with depth allocation -> user confirms
    |
[4] Generate <name>.md (evidence-based writing + PlantUML/D2 from symbol table)
    |  4.4 render diagrams (PNG/SVG)
    |  4.5 content self-check + Checkpoint 3: user confirms final md (with rendered diagrams)
    |
[5] Generate <name>.docx + artifact validation
    |
Output final artifacts
```

### 最终产物

| 产物               | 文件          | 说明                                           |
| ------------------ | ------------- | ---------------------------------------------- |
| 设计文档(Markdown) | `<name>.md`   | 含PlantUML/D2代码块的源文件（已通过质量自检） |
| 设计文档(Word)     | `<name>.docx` | 目录 + 编号标题 + 美化排版的正式文档（无封面） |

---

## 注意事项

1. 不要跳过Checkpoint：三个Checkpoint是强制的，闷头写的文档大概率推倒重来
2. 符号表是PlantUML图的唯一来源：PlantUML图中不出现符号表以外的元素。D2图的元素来源于代码中的实际标识符(rank编号、设备名、模块名等)，同样需要回查代码确认
3. 严格遵循模板：Markdown标题层级必须与所选模板完全一致
4. 不生成封面页：docx仅包含一个section（目录 + 正文）
5. 深度分配不均匀：重点章节至少是略写章节的3倍篇幅
6. 设计文档不是代码注释：重点描述设计决策和模块协作
7. 代码引用必须附带：每个架构描述附file:line
8. 中文优先：文档内容、图中标签均使用中文
9. 先定稿再转docx：md内容自检并经用户确认后才生成docx，但图表在格式化后即渲染，方便审稿
10. 排版一致性：所有元素必须使用统一样式
11. 领域探索增强: CANN项目应加载参考资料，并按1.5节的探索增强项和充分性检查增强执行
12. 中英文混排: 生成的Markdown应自行遵循中英文间距等排版规范，不依赖外部格式化工具
