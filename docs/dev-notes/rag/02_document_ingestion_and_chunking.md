# RAG 文档解析、清洗与 Chunking

## 1. 总体目标

Ingestion 的目标不是简单地“把文件切成固定长度文本”，而是：

> 尽可能恢复原始知识的结构，并生成适合 Retrieval 的知识单元。

推荐链路：

```text
Raw Input
   ↓
Document Loading
   ↓
Document Parsing
   ↓
Structured Document
   ↓
Normalization / Cleaning
   ↓
Hierarchical Chunking
   ↓
Chunks + Metadata
```

---

## 2. Document Loading 与 Parsing 的区别

### Loader

Loader 负责：

> 获取原始内容。

例如：

```text
PDF Loader
Markdown Loader
HTML Loader
XML Loader
Database Loader
API Loader
```

### Parser

Parser 负责：

> 理解内容结构并转换成统一 StructuredDocument。

因此：

```text
Loader
=
Read / Fetch

Parser
=
Understand / Structure
```

并不是所有输入都需要复杂 Parser。

Markdown、HTML、XML 本身已经包含较强结构，可以直接利用。

---

## 3. 主要输入场景

### 场景 A：有可靠文本层的 PDF

特点：

- 可以直接提取 Native Text
- 通常可以获得 Page、Position、Bounding Box 等信息
- 页面仍然包含字体、空白、多栏、表格、图片等视觉布局

核心任务：

- 恢复标题和 Section
- 恢复 Paragraph
- Table / Figure / Caption 识别
- Reading Order
- 多栏结构

原则：

> 能利用可靠 Native Text 时优先保留 Native Text，而不是为了识别文字重新 OCR。

可选工具：

- Docling
- Document VLM
- 其他 PDF Parser

---

### 场景 B：扫描 PDF / Image PDF

特点：

- 页面本质是图片
- 没有可靠 Native Text

需要恢复：

```text
Text
Layout
Reading Order
Table
Heading
Paragraph
Figure
Caption
```

可以使用：

- OCR + Layout Analysis
- Document VLM / OCR-VL
- 支持视觉文档理解的 Document Parser

对于 CvHub 这类希望降低自维护复杂度的系统，可以优先评估统一 Document Parser / Document VLM，但这属于工程策略，不是所有 RAG 系统的强制选择。

---

### 场景 C：Mixed PDF

例如：

```text
Page 1 → Native Text
Page 2 → Native Text
Page 3 → Scan
Page 4 → Scan
Page 5 → Native Text
```

不能简单将整个 PDF 二分为：

```text
Text PDF
or
Scan PDF
```

更合理的是支持：

```text
Page-level
or
Block-level
```

判断和 fallback。

最终所有页面统一输出：

```text
StructuredDocument
```

---

### 场景 D：有完整结构的文本

例如：

- Markdown
- HTML
- XML
- 已经结构化的文本

原则：

> 已有可靠结构时优先使用现有结构，不需要模型重新猜测。

可以直接使用：

- Heading Split
- Section Split
- Paragraph Split
- HTML Structure Split
- XML Structure Split

---

### 场景 E：无可靠结构的纯文本

特点：

```text
只有连续文本
没有 Heading
没有 Section
视觉信息已经丢失
```

这时才需要更多语义方法：

- Sentence Splitting
- Semantic Segmentation
- Embedding-based Segmentation
- LLM Segmentation
- Topic Segmentation

---

## 4. Structured Document

Parser 的目标输出应该尽可能统一。

例如：

```text
Document
├── Title
├── Section
│   ├── Heading
│   ├── Paragraph
│   ├── Paragraph
│   └── Table
└── Section
    ├── Paragraph
    ├── Figure
    └── Caption
```

典型 Block 类型：

```text
Title
Heading
Paragraph
List
Table
Figure
Caption
Formula
Code
```

典型结构信息：

```text
page
bbox
reading_order
parent
section
heading_path
```

---

## 5. Normalization / Cleaning

Parser 输出后不建议直接进入 Chunker。

完整流程：

```text
Parser
 ↓
Raw Structured Document
 ↓
Normalization / Cleaning
 ↓
Clean Structured Document
 ↓
Chunker
```

这一层主要处理：

- Header / Footer 去除
- Page Number 去除
- Watermark 过滤
- 重复内容去除
- 空白清理
- 异常字符清理
- Encoding 修复
- Reading Order 修复
- 跨页 Paragraph 合并
- Block Continuity
- Metadata 统一

这部分优先使用确定性规则。

---

## 6. Chunk 是什么

Chunk 是 Retrieval 的基本文本单元。

```text
Document
   ↓
Chunk 1
Chunk 2
Chunk 3
...
```

Chunking 的目标不是单纯控制长度，而是同时考虑：

```text
Semantic Completeness
+
Retrieval Precision
+
Context Completeness
+
Token Cost
```

---

## 7. 常见 Chunking 方法

### Fixed-size Chunking

按照：

- Token 数量
- Character 数量

切分。

优点：

- 简单
- 快
- 稳定

缺点：

- 容易破坏语义结构

适合：

- fallback
- 结构非常弱的数据
- 最终 size control

---

### Recursive Chunking

按照一组优先级逐级切分：

```text
Section
 ↓
Paragraph
 ↓
Sentence
 ↓
Token
```

直到满足 Size Limit。

比纯 Fixed-size 更合理。

---

### Structure-aware Chunking

利用：

- Heading
- Section
- Paragraph
- List
- Table
- Document Block

进行切分。

对于结构化文档通常应该优先采用。

---

### Semantic Chunking

根据语义变化寻找 Chunk Boundary。

典型流程：

```text
Text
 ↓
Sentence Split
 ↓
Sentence Embedding
 ↓
Compare Adjacent Semantics
 ↓
Detect Topic Change
 ↓
Segment
```

适用于：

- 长 Section
- 连续文本
- 缺少可靠子结构

---

### LLM-based Chunking

使用 LLM 判断：

- Topic Boundary
- Logic Boundary
- Content Transition

优点：

- 语义理解强

缺点：

- 成本高
- 延迟高
- 输出不完全确定
- 需要验证

因此更适合作为：

> Conditional Step / Fallback

而不是默认处理所有文档。

---

## 8. 推荐的 Hierarchical Chunking

更合理的策略不是选择一种 Chunker，而是：

```text
Structured Content
       ↓
Structure First
       ↓
Section / Paragraph / Table
       ↓
Size Check
   ┌───────┼────────┐
   │       │        │
 Too Big  Good    Too Small
   │                │
Split Further      Merge
   │                │
   └────────┬───────┘
            ↓
       Final Chunk
```

### Too Big

例如：

```text
Section = 5000 tokens
```

继续：

- Paragraph Split
- Semantic Split
- Sentence Split
- Token Limit Split

### Too Small

例如：

```text
Heading + 30 token paragraph
```

可以：

- 与 Parent Heading 合并
- 与相邻 Paragraph 合并
- 添加 Section Context

---

## 9. Chunk Overlap

固定 Chunking 中常见：

```text
Chunk 1: token 0-500
Chunk 2: token 450-950
```

Overlap 可以减少边界导致的上下文丢失。

但 Overlap 过大会造成：

- 重复索引
- Retrieval Duplicate
- Storage 增加
- Context 重复

因此：

> Overlap 是补偿边界问题的工具，不应该替代良好的结构化 Chunking。

---

## 10. 特殊 Block

### Table

表格不能简单转换成无结构连续文本。

需要尽量保留：

```text
Header
Row
Column
Cell Relation
Caption
```

可以使用：

- Markdown Table
- Structured JSON
- Table-specific representation

Table 应被视为特殊 Block。

### Figure / Diagram / Chart

部分信息只存在视觉内容中。

可考虑：

- 保留 Figure Block
- 保留 Caption
- VLM 生成 Description
- Multimodal Retrieval

基础 Text RAG 可以先将重要视觉内容转换为结构化文字描述。

### Formula

应尽量：

- 保持 Formula Block 完整
- 与相关 Paragraph / Caption 保持关联

### Code

代码不适合普通 Sentence Split。

应：

- 保留 Code Block
- 对超长代码使用 Code-aware Split

---

## 11. 跨页内容

常见：

- Paragraph 跨页
- Table 跨页
- List 跨页
- Caption 跨页

因此：

```text
Page Boundary
≠
Semantic Boundary
```

Chunking 前应该进行：

```text
Block Continuity Detection
+
Cross-page Merge
```

---

## 12. Metadata

Chunk 建议保留：

```text
chunk_id
document_id
document_version
page
section
heading_path
chunk_type
language
source
bbox
created_at
```

可以进一步区分：

### Document-level Metadata

例如：

```text
document_id
version
source
language
created_at
permissions
```

### Chunk-level Metadata

例如：

```text
chunk_id
page
heading_path
chunk_type
bbox
parent_chunk_id
```

Metadata 用于：

- Filter
- Citation
- Debugging
- Versioning
- Access Control
- Evaluation

---

## 13. Parent-Child / Hierarchical Chunk

可以同时维护不同粒度：

```text
Document
 ↓
Section
 ↓
Paragraph Chunk
```

例如：

> 使用较小 Chunk 做精确 Retrieval，但最终返回它所属的较大 Parent Section。

这类方法通常称为：

```text
Parent-Child Retrieval
Small-to-Big Retrieval
```

核心思想：

```text
Small Chunk
=
Retrieval Precision

Large Parent
=
Context Completeness
```

---

## 14. 多语言

语言会影响：

- Sentence Splitting
- Tokenization
- Semantic Segmentation
- Embedding

建议保留：

```text
language metadata
```

并使用：

- Multilingual Sentence Splitter
- Multilingual Embedding Model

最终效果应使用真实 Query / Document Evaluation。

---

## 15. 文档版本

Knowledge Base 可能存在：

```text
Document v1
Document v2
Document v3
```

应考虑：

```text
document_id
version
active / inactive
updated_at
```

更新时需要：

```text
Delete / Deactivate Old Chunks
 ↓
Parse New Version
 ↓
Re-chunk
 ↓
Re-embed
 ↓
Re-index
```

---

## 16. Parsing / Chunking Evaluation

不能只根据感觉选择：

```text
500 tokens
1000 tokens
Semantic Chunking
Docling
Document VLM
```

应该通过真实 Query 测试：

```text
Parsing Strategy
      +
Chunk Strategy
      ↓
Retrieval
      ↓
Evaluation
```

常见 Retrieval 指标：

- Recall@K
- Precision@K
- Hit Rate
- MRR
- nDCG

最终原则：

> **Chunk 的质量应该通过 Retrieval 效果判断，而不是仅通过 Chunk 本身是否“看起来完整”。**
