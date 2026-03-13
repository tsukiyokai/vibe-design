# D2示意图编写指南

D2用于绘制非UML示意图：拓扑布局、数据流、架构总览。与PlantUML互补，不替代。

## 安装与渲染

```bash
# macOS
brew install d2

# Linux
curl -fsSL https://d2lang.com/install.sh | sh -s --

# 渲染为SVG
d2 --theme 200 input.d2 output.svg

# 语法检查
d2 fmt --check input.d2
```

---

## 核心语法速查

### 节点与连接

```d2
# 节点(自动推断形状)
server: Application Server

# 连接(->单向, <->双向, --->虚线)
client -> server: HTTP request
server -> db: SQL query
server <-> cache: read/write
client ---> cdn: static assets
```

### 容器(嵌套分组)

```d2
# 容器 = 带花括号的节点
pod0: Pod 0 {
  node0: Node 0 {
    dev0: Device 0
    dev1: Device 1
  }
  node1: Node 1 {
    dev2: Device 2
    dev3: Device 3
  }
}

# 跨容器连接
pod0.node0.dev0 -> pod0.node1.dev2: HCCS {
  style.stroke: "#E87D3E"
}
```

### Grid布局

```d2
# grid-rows或grid-columns控制排列
rank_matrix: Rank Matrix {
  grid-rows: 2
  grid-columns: 4

  r0: Rank 0
  r1: Rank 1
  r2: Rank 2
  r3: Rank 3
  r4: Rank 4
  r5: Rank 5
  r6: Rank 6
  r7: Rank 7
}
```

### 样式

```d2
# 节点样式
server: Server {
  style: {
    fill: "#E8F4FD"
    stroke: "#2E5984"
    border-radius: 8
    font-color: "#1A3A5C"
  }
}

# 连接样式
a -> b: data {
  style: {
    stroke: "#E87D3E"
    stroke-width: 2
    stroke-dash: 3    # dash = dashed line
  }
}
```

### 形状

```d2
proc: Process {shape: rectangle}
decision: Check? {shape: diamond}
store: Database {shape: cylinder}
queue: Queue {shape: queue}
cloud: External {shape: cloud}
pkg: Package {shape: package}
```

### 图标与标签

```d2
# tooltip(悬停显示)
server: Server {
  tooltip: handles 10k req/s
}

# 多行标签
desc: |md
  **Title**
  - item 1
  - item 2
|
```

---

## 全局样式规范

与PlantUML保持一致的蓝灰色系：

```d2
# 在.d2文件顶部定义
style: {
  fill: "#FFFFFF"
}

# 色彩规范(与plantuml-guide.md一致)
# 主色:      #2E5984 (深蓝, 用于边框/标题)
# 次要:      #4A90D9 (中蓝, 用于连接线)
# 背景:      #E8F4FD (浅蓝, 用于容器填充)
# 强调:      #E87D3E (橙色, 用于关键路径/高亮)
# 文字:      #1A3A5C (深色, 用于标签)
# 次要文字:  #6B7B8D (灰色, 用于注释)
```

渲染时使用 `--theme 200`(Neutral default)保持简洁。

---

## CANN场景示例

### 示例1: Pod内Rank拓扑

8卡设备的rank分布和通信链路：

```d2
direction: right

pod: Pod (8 ranks) {
  node0: Node 0 {
    style.fill: "#E8F4FD"
    style.stroke: "#2E5984"

    grid-rows: 1
    dev0: "Device 0\nRank 0" {
      style.fill: "#FFFFFF"
      style.stroke: "#4A90D9"
    }
    dev1: "Device 1\nRank 1" {
      style.fill: "#FFFFFF"
      style.stroke: "#4A90D9"
    }
    dev2: "Device 2\nRank 2" {
      style.fill: "#FFFFFF"
      style.stroke: "#4A90D9"
    }
    dev3: "Device 3\nRank 3" {
      style.fill: "#FFFFFF"
      style.stroke: "#4A90D9"
    }
  }

  node1: Node 1 {
    style.fill: "#E8F4FD"
    style.stroke: "#2E5984"

    grid-rows: 1
    dev4: "Device 4\nRank 4" {
      style.fill: "#FFFFFF"
      style.stroke: "#4A90D9"
    }
    dev5: "Device 5\nRank 5" {
      style.fill: "#FFFFFF"
      style.stroke: "#4A90D9"
    }
    dev6: "Device 6\nRank 6" {
      style.fill: "#FFFFFF"
      style.stroke: "#4A90D9"
    }
    dev7: "Device 7\nRank 7" {
      style.fill: "#FFFFFF"
      style.stroke: "#4A90D9"
    }
  }

  # L1: intra-node (HCCS)
  node0.dev0 -> node0.dev1: L1 HCCS {style.stroke: "#4A90D9"}
  node0.dev1 -> node0.dev2: L1 HCCS {style.stroke: "#4A90D9"}
  node0.dev2 -> node0.dev3: L1 HCCS {style.stroke: "#4A90D9"}

  # L2: inter-node (RDMA)
  node0.dev3 -> node1.dev4: L2 RDMA {
    style.stroke: "#E87D3E"
    style.stroke-width: 2
  }
}
```

### 示例2: 通信层级(L0/L1/L2)

三层通信层级的数据搬运路径：

```d2
direction: down

l0: L0 - Intra-Device {
  style.fill: "#E8F4FD"
  style.stroke: "#2E5984"
  desc: "DMA copy within device\nBandwidth: highest\nLatency: lowest"
  aicore: AI Core
  ub: Unified Buffer
  gm: Global Memory
  aicore -> ub: DMA {style.stroke: "#2E5984"}
  ub -> gm: DMA {style.stroke: "#2E5984"}
}

l1: L1 - Intra-Node {
  style.fill: "#F0F4E8"
  style.stroke: "#5A8432"
  desc: "HCCS/PCIe between devices\nNUMA-aware"
  dev0: Device 0
  dev1: Device 1
  dev0 <-> dev1: HCCS {style.stroke: "#5A8432"}
}

l2: L2 - Inter-Node {
  style.fill: "#FDF0E8"
  style.stroke: "#E87D3E"
  desc: "RDMA/TCP over network\nBandwidth allocation needed"
  node0: Node 0
  node1: Node 1
  node0 <-> node1: RDMA {style.stroke: "#E87D3E"}
}

l0 -> l1: "notify/sync" {style.stroke-dash: 3}
l1 -> l2: "notify/sync" {style.stroke-dash: 3}
```

### 示例3: 流水线阶段与同步点

DataCopy/Compute流水线的barrier/flag配对：

```d2
direction: right

stage1: Stage 1 - DataCopy In {
  style.fill: "#E8F4FD"
  style.stroke: "#2E5984"
  op: "DataCopy\nGM -> UB"
}

barrier1: PipeBarrier {
  shape: diamond
  style.fill: "#E87D3E"
  style.font-color: "#FFFFFF"
}

stage2: Stage 2 - Compute {
  style.fill: "#E8F4FD"
  style.stroke: "#2E5984"
  op: "Compute\non UB data"
}

barrier2: PipeBarrier {
  shape: diamond
  style.fill: "#E87D3E"
  style.font-color: "#FFFFFF"
}

stage3: Stage 3 - DataCopy Out {
  style.fill: "#E8F4FD"
  style.stroke: "#2E5984"
  op: "DataCopy\nUB -> GM"
}

stage1 -> barrier1: "SetFlag(pipe=DMA)" {style.stroke: "#4A90D9"}
barrier1 -> stage2: "WaitFlag(pipe=DMA)" {style.stroke: "#4A90D9"}
stage2 -> barrier2: "SetFlag(pipe=Compute)" {style.stroke: "#4A90D9"}
barrier2 -> stage3: "WaitFlag(pipe=Compute)" {style.stroke: "#4A90D9"}
```

### 示例4: Selector-Executor-Template分派链

hccl算子的分派架构总览：

```d2
direction: down

op_entry: "AllReduceOperator\n(op entry)" {
  style.fill: "#E8F4FD"
  style.stroke: "#2E5984"
  style.font-color: "#1A3A5C"
}

selector: Selector Layer {
  style.fill: "#F5F5F5"
  style.stroke: "#6B7B8D"

  sel: "AllReduceSelector" {style.stroke: "#4A90D9"}
  sel_logic: "Select by:\n- dataSize\n- rankSize\n- DevType" {
    style.fill: "#FFFFFF"
    style.stroke-dash: 3
  }
}

executor: Executor Layer {
  style.fill: "#F5F5F5"
  style.stroke: "#6B7B8D"

  grid-rows: 1
  exec_ring: "RingAllReduce\nExecutor" {style.stroke: "#4A90D9"}
  exec_hd: "HalvingDoubling\nExecutor" {style.stroke: "#4A90D9"}
  exec_mesh: "MeshAllReduce\nExecutor" {style.stroke: "#4A90D9"}
}

template: Template Layer {
  style.fill: "#F5F5F5"
  style.stroke: "#6B7B8D"

  grid-rows: 1
  tpl_ring: "RingTemplate" {style.stroke: "#2E5984"}
  tpl_hd: "HDTemplate" {style.stroke: "#2E5984"}
  tpl_mesh: "MeshTemplate" {style.stroke: "#2E5984"}
}

op_entry -> selector.sel: "dispatch" {style.stroke: "#2E5984"}
selector.sel -> executor.exec_ring: "ring" {style.stroke: "#4A90D9"}
selector.sel -> executor.exec_hd: "halving-doubling" {style.stroke: "#4A90D9"}
selector.sel -> executor.exec_mesh: "mesh" {style.stroke: "#4A90D9"}
executor.exec_ring -> template.tpl_ring {style.stroke: "#6B7B8D"}
executor.exec_hd -> template.tpl_hd {style.stroke: "#6B7B8D"}
executor.exec_mesh -> template.tpl_mesh {style.stroke: "#6B7B8D"}
```

### 示例5: 两级通信层级映射矩阵

N对M映射关系用矩阵表达，避免连线交叉。行=Level 0子组，列=Level 1 slot，交叉点=rank编号：

```d2
direction: down

rank-mapping: "Rank Mapping Matrix  (instSizeList=[8,4], GCD=4)" {
  grid-rows: 4
  grid-columns: 5

  # Header row: Level 1 slot labels (dashed border)
  corner: "Level 0 \\ Level 1" {
    style.font-size: 13
    style.stroke-dash: 5
  }
  h1: "Slot 0\nrankId%4=0" {
    style.stroke-dash: 5
    style.font-size: 14
  }
  h2: "Slot 1\nrankId%4=1" {
    style.stroke-dash: 5
    style.font-size: 14
  }
  h3: "Slot 2\nrankId%4=2" {
    style.stroke-dash: 5
    style.font-size: 14
  }
  h4: "Slot 3\nrankId%4=3" {
    style.stroke-dash: 5
    style.font-size: 14
  }

  # Row 0: Subgroup 0 (Server A, ranks 0-3)
  sg0: "Subgroup 0\nServer A [0..3]" {
    style.stroke-dash: 3
    style.font-size: 13
  }
  r0: Rank 0
  r1: Rank 1
  r2: Rank 2
  r3: Rank 3

  # Row 1: Subgroup 1 (Server A, ranks 4-7)
  sg1: "Subgroup 1\nServer A [4..7]" {
    style.stroke-dash: 3
    style.font-size: 13
  }
  r4: Rank 4
  r5: Rank 5
  r6: Rank 6
  r7: Rank 7

  # Row 2: Subgroup 2 (Server B, ranks 8-11)
  sg2: "Subgroup 2\nServer B [8..11]" {
    style.stroke-dash: 3
    style.font-size: 13
  }
  r8: Rank 8
  r9: Rank 9
  r10: Rank 10
  r11: Rank 11
}
```

对比反模式（连线图）：12个rank到4个slot的连线会产生大量交叉，可读性极差。矩阵布局零交叉，信息密度更高。

---

## 编写约束

1. 图中的设备名、rank编号、模块名必须与代码/配置一致
2. 用D2注释标注来源：`# source: file:line`
3. 连接标签标注通信层级(L0/L1/L2)或数据量
4. 容器嵌套层级不超过4层(pod→node→device→rank)
5. 单张图节点数不超过30个，超过则拆分为多张
6. 绘制完成后逐元素回查代码确认
7. N对M映射用矩阵，不用连线：当两个维度存在映射关系（如rank→slot、子组→层级）时，用grid布局（行=维度A，列=维度B，交叉点=映射结果），不要画逐条连线。连线图在节点>8时因交叉而不可读
8. 宽高比必须在4:3到3:4之间，渲染后用图片尺寸验证，超过3:1或1:3立即返工。常见导致狭长图的反模式及修复：

| 反模式 | 效果 | 修复 |
|--------|------|------|
| 容器内`grid-rows: 1` + 元素多 | 所有元素水平铺开，图极宽 | 增大grid-rows使元素折行(如8个元素用grid-rows: 2) |
| `direction: right` + 深层容器嵌套 | 容器沿水平堆叠，宽度累加 | 改为`direction: down`或拆分为多张图 |
| `direction: down` + 容器内又`direction: right` | 内外方向冲突导致一个轴失控拉伸 | 统一内外方向，或内层用grid替代direction |
| 大量跨容器连线 | D2自动布局为连线预留空间导致图膨胀 | 用矩阵布局替代连线(见约束7) |

## 何时选D2而非PlantUML

| 场景 | 选D2 | 选PlantUML |
|------|------|------------|
| 代码结构(类/接口/继承) | | x |
| 调用时序(函数A调B调C) | | x |
| 状态机/活动流程 | | x |
| 设备拓扑/rank排列 | x | |
| 通信层级/数据搬运路径 | x | |
| 流水线阶段与同步点 | x | |
| 架构总览/分派链 | x | |
| 部署视图/物理布局 | x | |
