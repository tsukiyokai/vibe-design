# docx排版规范

本文档定义了设计文档转 .docx时的完整排版规范。使用docx-js (`npm install -g docx`)生成。

## 页面设置

| 项目       | 规格                                                          |
| ---------- | ------------------------------------------------------------- |
| 纸张       | A4 (11906 x 16838 DXA)                                        |
| 页边距     | 上下1440 DXA（1英寸），左右1440 DXA                           |
| 正文区宽度 | 9026 DXA                                                      |
| 页眉       | 右对齐，灰色小字：项目名 + "设计文档"（如 "XXX软件设计文档"） |
| 页脚       | 居中显示页码                                                  |

## 目录页

文档第一页为目录页（不生成封面页），使用 `TableOfContents`，需配合heading styles的 `outlineLevel`。

docx-js生成的TOC只包含域代码，不含实际条目。必须在Document配置中启用`updateFields`，Word打开时会自动刷新TOC：

```javascript
const doc = new Document({
  features: {
    updateFields: true  // Word打开时自动更新TOC
  },
  // ...其他配置
});

// TOC定义
new TableOfContents("目  录", {
  hyperlink: true,
  headingStyleRange: "1-3"
})
```

## 标题体系

采用带编号的层级标题，通过docx-js的numbering配置实现自动编号：

| 级别 | 字体       | 字号              | 颜色            | 编号格式      | 用途       |
| ---- | ---------- | ----------------- | --------------- | ------------- | ---------- |
| H1   | Arial Bold | 28pt (56 half-pt) | #1A3C5E（深蓝） | `1` / `2`     | 文档大部分 |
| H2   | Arial Bold | 16pt (32 half-pt) | #2E5984（蓝）   | `1.1` / `1.2` | 章节       |
| H3   | Arial Bold | 13pt (26 half-pt) | #333333（深灰） | `1.1.1`       | 子章节     |
| H4   | Arial Bold | 11pt (22 half-pt) | #333333         | 无编号        | 段落标题   |

```javascript
// 标题编号配置
{
  reference: "heading-numbering",
  levels: [
    { level: 0, format: LevelFormat.DECIMAL, text: "%1",
      alignment: AlignmentType.LEFT,
      style: { paragraph: { indent: { left: 0, hanging: 0 } } } },
    { level: 1, format: LevelFormat.DECIMAL, text: "%1.%2",
      alignment: AlignmentType.LEFT,
      style: { paragraph: { indent: { left: 0, hanging: 0 } } } },
    { level: 2, format: LevelFormat.DECIMAL, text: "%1.%2.%3",
      alignment: AlignmentType.LEFT,
      style: { paragraph: { indent: { left: 0, hanging: 0 } } } },
  ]
}

// Heading 样式 (覆盖内置样式)
paragraphStyles: [
  { id: "Heading1", name: "Heading 1", basedOn: "Normal", next: "Normal", quickFormat: true,
    run: { size: 56, bold: true, font: "Arial", color: "1A3C5E" },
    paragraph: { spacing: { before: 480, after: 240 }, outlineLevel: 0,
                 numbering: { reference: "heading-numbering", level: 0 } } },
  { id: "Heading2", name: "Heading 2", basedOn: "Normal", next: "Normal", quickFormat: true,
    run: { size: 32, bold: true, font: "Arial", color: "2E5984" },
    paragraph: { spacing: { before: 320, after: 160 }, outlineLevel: 1,
                 numbering: { reference: "heading-numbering", level: 1 } } },
  { id: "Heading3", name: "Heading 3", basedOn: "Normal", next: "Normal", quickFormat: true,
    run: { size: 26, bold: true, font: "Arial", color: "333333" },
    paragraph: { spacing: { before: 240, after: 120 }, outlineLevel: 2,
                 numbering: { reference: "heading-numbering", level: 2 } } },
]
```

## 正文样式

| 项目     | 规格                      |
| -------- | ------------------------- |
| 字体     | Arial（英文），中文随系统 |
| 字号     | 11pt (22 half-pt)         |
| 行距     | 1.15倍(line: 276)         |
| 段后间距 | 120 DXA                   |

```javascript
default: {
  document: {
    run: { font: "Arial", size: 22 },
    paragraph: { spacing: { after: 120, line: 276 } }
  }
}
```

## 表格样式

所有表格采用统一的专业配色：

| 部件               | 样式                                 |
| ------------------ | ------------------------------------ |
| 表头行             | 背景 #2E5984（深蓝），字色白色，加粗 |
| 奇数行(1, 3, 5...) | 背景 #FFFFFF（白色）                 |
| 偶数行(2, 4, 6...) | 背景 #F2F7FB（浅蓝灰）               |
| 边框               | #D6E4EF（蓝灰），1pt                 |
| 单元格内边距       | top/bottom: 80, left/right: 120 DXA  |
| 表格宽度           | 固定9026 DXA（占满正文区）           |

```javascript
const tblBorder = { style: BorderStyle.SINGLE, size: 1, color: "D6E4EF" };
const tblBorders = { top: tblBorder, bottom: tblBorder, left: tblBorder, right: tblBorder };
const tblCellMargins = { top: 80, bottom: 80, left: 120, right: 120 };

function tblHeaderCell(text, width) {
  return new TableCell({
    borders: tblBorders,
    width: { size: width, type: WidthType.DXA },
    shading: { fill: "2E5984", type: ShadingType.CLEAR },
    margins: tblCellMargins,
    children: [new Paragraph({ children: [
      new TextRun({ text, bold: true, font: "Arial", size: 20, color: "FFFFFF" })
    ] })]
  });
}

function tblCell(text, width, isEvenRow) {
  return new TableCell({
    borders: tblBorders,
    width: { size: width, type: WidthType.DXA },
    shading: isEvenRow ? { fill: "F2F7FB", type: ShadingType.CLEAR } : undefined,
    margins: tblCellMargins,
    children: [new Paragraph({ children: [
      new TextRun({ text, font: "Arial", size: 20 })
    ] })]
  });
}
```

## 代码块样式

代码区域需要清晰区分于正文：

| 项目      | 规格                        |
| --------- | --------------------------- |
| 字体      | Consolas, 9pt (18 half-pt)  |
| 背景      | #F5F7FA（浅灰）             |
| 左边框    | 3pt, #2E5984（蓝色竖条）    |
| 段前/段后 | 60 DXA                      |
| 行距      | 单倍行距(line: 240)         |

```javascript
function codeBlock(text) {
  return new Paragraph({
    spacing: { before: 60, after: 60, line: 240 },
    shading: { fill: "F5F7FA", type: ShadingType.CLEAR },
    border: { left: { style: BorderStyle.SINGLE, size: 6, color: "2E5984", space: 8 } },
    indent: { left: 240 },
    children: [new TextRun({ text, font: "Consolas", size: 18 })]
  });
}

// 多行代码块：每行一个 Paragraph
function codeBlockMulti(lines) {
  return lines.map(line => codeBlock(line));
}
```

## 行内代码样式

正文中的行内代码使用与代码块相同的字体，配浅灰背景：

| 项目   | 规格                       |
| ------ | -------------------------- |
| 字体   | Consolas, 10pt (20 half-pt)|
| 背景   | #F5F7FA（浅灰）            |

```javascript
function inlineCode(text) {
  return new TextRun({
    text,
    font: "Consolas",
    size: 20,
    shading: { fill: "F5F7FA", type: ShadingType.CLEAR }
  });
}
```

## 图片样式

| 项目      | 规格                                       |
| --------- | ------------------------------------------ |
| 对齐      | 居中                                       |
| 最大宽度  | 正文区的90% （约560px / 8123 DXA）         |
| 段前/段后 | 200 DXA                                    |
| 图注      | 图片下方居中，灰色9pt，格式："图N: <标题>" |

```javascript
let figureCount = 0;

function figureWithCaption(filePath, width, height, title) {
  figureCount++;
  return [
    new Paragraph({
      spacing: { before: 200, after: 80 },
      alignment: AlignmentType.CENTER,
      children: [new ImageRun({
        type: "png", data: fs.readFileSync(filePath),
        transformation: { width, height },
        altText: { title, description: title, name: title }
      })]
    }),
    new Paragraph({
      spacing: { after: 200 },
      alignment: AlignmentType.CENTER,
      children: [new TextRun({
        text: `图 ${figureCount}: ${title}`,
        font: "Arial", size: 18, color: "666666", italics: true
      })]
    })
  ];
}
```

## 列表样式

```javascript
numbering: {
  config: [
    // 有序列表
    { reference: "ordered",
      levels: [
        { level: 0, format: LevelFormat.DECIMAL, text: "%1.",
          alignment: AlignmentType.LEFT,
          style: { paragraph: { indent: { left: 720, hanging: 360 } } } },
        { level: 1, format: LevelFormat.LOWER_LETTER, text: "%2)",
          alignment: AlignmentType.LEFT,
          style: { paragraph: { indent: { left: 1080, hanging: 360 } } } },
      ] },
    // 无序列表
    { reference: "bullets",
      levels: [
        { level: 0, format: LevelFormat.BULLET, text: "\u2022",
          alignment: AlignmentType.LEFT,
          style: { paragraph: { indent: { left: 720, hanging: 360 } } } },
        { level: 1, format: LevelFormat.BULLET, text: "\u25E6",
          alignment: AlignmentType.LEFT,
          style: { paragraph: { indent: { left: 1080, hanging: 360 } } } },
      ] },
  ]
}
```

注意：每个独立编号序列需使用不同的 `reference` 名称（如 `"ordered1"`, `"ordered2"`），否则编号会延续上一个序列。

## 完整文档结构

生成的docx文件不含封面页，只有一个section（含页眉页脚），标题层级严格匹配所选模板：

```
Section 1: body (with header/footer)
  +-- TOC (TableOfContents)
  +-- [PageBreak]
  +-- H1: Part 1
  |   +-- H2: Chapter
  |   |   +-- H3: Section
  |   |   |   +-- H4: Subsection
  |   |   ...
  |   +-- [PageBreak]
  +-- H1: Part 2
      +-- ...
```

标题层级必须严格匹配所选模板，不可增加或跳过层级。
