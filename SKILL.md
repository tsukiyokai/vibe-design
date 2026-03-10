---
name: vibe-design
description: "用于生成软件设计文档。当用户提到 '设计文档'、'design doc'、'SD'、'SRS'、'需求分析'、'软件设计'、'架构设计'，或要求根据需求描述生成技术设计文档时触发。自动学习代码仓库中的相关代码和文档，基于模板生成 markdown 格式的设计文档（含 4+1 架构视图的 PlantUML 图），并转换为 .docx 文件。"
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
| 4. 生成Markdown | 按写作准则生成设计文档（含PlantUML图）       | `<name>.md`       |
| 5. PlantUML渲染 | 提取 → 编译PNG → 更新引用                    | PNG图片           |
| 6. 格式转换     | 生成 .docx + 质量自检                        | `<name>.docx`     |

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

---

## 第二步：知识对齐(Checkpoint 1)

探索完成后，必须向用户呈现知识摘要，确认理解正确后才能继续。

呈现内容：
- 模块职责：每个相关模块做什么
- 核心接口：关键的类、函数、数据结构
- 模块交互：谁调用谁、数据怎么流动
- 设计意图：为什么现有代码这样设计（如果能判断的话）
- 盲区标注：明确列出"我不确定"或"没找到"的部分

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

要求（正模式）：
- 先结论再展开：这个模块解决什么问题 → 怎么解决的 → 为什么选这个方案
- 每个架构描述附带代码引用(file:line)，让读者能追溯
- 站在"向新人解释这个系统"的视角，而非"向编译器解释代码"
- 设计决策要说明取舍：选了A方案，B方案为什么不行

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

### 4.3 其他写作要求

- 所有模板中定义的章节必须存在且非空
- 根据需求选用2-4个最相关的4+1架构视图
- 中文优先：文档内容、图中标签均使用中文
- 文件名从需求标题派生，如 `自定义算子支持aclGraph-ccu模式.md`

---

## 第五步：PlantUML渲染

### 5.1 安装检查

```bash
# macOS
brew install plantuml

# Linux
apt-get install -y plantuml
# 或手动下载 jar
curl -L -o /tmp/plantuml.jar https://github.com/plantuml/plantuml/releases/latest/download/plantuml.jar
```

### 5.2 提取与渲染

1. 创建临时目录：`mkdir -p /tmp/designdoc_puml`
2. 从markdown中提取每个plantuml代码块为 `.puml` 文件（文件名用 `@startuml` 后的标识符）
3. 批量渲染：`plantuml -tpng -o /tmp/designdoc_puml/ /tmp/designdoc_puml/*.puml`
4. 更新markdown中的图片引用：将plantuml代码块替换为 `![标题](path.png)`

详细的PlantUML编写规范和示例参见 [plantuml-guide.md](references/plantuml-guide.md)。

---

## 第六步：格式美化与Word转换

### 6.1 生成 .docx

使用docx-js (`npm install -g docx`)按排版规范生成 .docx文件。完整的排版规范、样式定义和代码示例参见 [docx-formatting.md](references/docx-formatting.md)。

核心要求：
- 不生成封面页，仅目录 + 正文
- 带编号的层级标题(H1 ~ H4)
- 统一的表格、代码块、图片、列表样式
- 页眉页脚（页眉右对齐项目名称，页脚居中页码）
- 标题层级必须严格匹配所选模板

### 6.2 质量自检（必须执行）

文档生成完成后，执行以下检查。发现问题直接修改原Markdown文件。

文档自检：

| 检查项         | 规则                                                  | 修复方式                   |
| -------------- | ----------------------------------------------------- | -------------------------- |
| 必填章节完整性 | 模板定义的所有章节均必须存在且非空                    | 补充缺失章节内容           |
| 表格完整性     | 所有表格不得有空单元格，表头与数据行列数一致          | 填充空单元格或修正列数     |
| PlantUML可编译 | 所有 `.puml` 文件必须通过 `plantuml -syntax` 语法检查 | 修复语法错误后重新渲染     |
| 代码块语言标注 | 所有代码块必须标注语言                                | 给裸代码块补上语言标注     |
| 图片引用有效   | 所有 `![...](path)` 引用的图片文件必须存在            | 重新渲染缺失图片或修正路径 |

与代码一致性校验：

| 检查项         | 方法                           | 修复方式             |
| -------------- | ------------------------------ | -------------------- |
| 文件路径有效   | 用Glob验证文档中引用的路径存在 | 修正为实际路径       |
| 函数签名一致   | Grep验证函数签名存在且一致     | 修正签名为代码实际值 |
| 枚举/宏值正确  | 对比文档引用值与代码定义       | 修正为代码实际值     |
| 结构体字段一致 | 对比文档中的字段列表与代码定义 | 增删字段使其一致     |

若自检修改了Markdown，需重新生成 .docx。

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
[4] Generate <name>.md (evidence-based writing + PlantUML from symbol table)
    |
[5] Extract PlantUML -> .puml -> render PNG -> update image refs
    |
[6] Generate <name>.docx (TOC + numbered headings + styled tables)
    |
[7] Quality check -> fix issues in .md -> regenerate .docx if changed
    |
Output final artifacts
```

### 最终产物

| 产物               | 文件          | 说明                                           |
| ------------------ | ------------- | ---------------------------------------------- |
| 设计文档(Markdown) | `<name>.md`   | 含PlantUML代码块的源文件（已通过质量自检）     |
| 设计文档(Word)     | `<name>.docx` | 目录 + 编号标题 + 美化排版的正式文档（无封面） |

---

## 注意事项

1. 不要跳过Checkpoint：两个Checkpoint是强制的，闷头写的文档大概率推倒重来
2. 符号表是PlantUML的唯一来源：图中不出现符号表以外的元素
3. 严格遵循模板：Markdown标题层级必须与所选模板完全一致
4. 不生成封面页：docx仅包含一个section（目录 + 正文）
5. 深度分配不均匀：重点章节至少是略写章节的3倍篇幅
6. 设计文档不是代码注释：重点描述设计决策和模块协作
7. 代码引用必须附带：每个架构描述附file:line
8. 中文优先：文档内容、图中标签均使用中文
9. 质量自检必须执行：不可跳过第六步的检查环节
10. 排版一致性：所有元素必须使用统一样式
