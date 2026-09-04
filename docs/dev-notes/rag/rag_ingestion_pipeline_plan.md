# RAG Ingestion Pipeline 规划

## 1. 设计目标

计划中的 Pipeline 不追求引入大量不同模型，而是保持组件少、职责清晰、容易替换和评估。

核心设计原则：

> **一个统一 Document Parser + 规则优先的结构处理 + 条件式语义分割 + 分层 Chunking**

模型只用于真正需要模型理解的部分。

---

## 2. 总体 Pipeline

```text
Input
  ↓
Document / Text Detection
  ↓
Parser / Segmenter
  ↓
Structured Content
  ↓
Normalization / Cleaning
  ↓
Hierarchical Chunker
  ↓
Final Chunks
  ↓
Embedding
  ↓
Vector Store
```

---

## 3. Parser 层

文档类输入只选择一个主要 Document Parser：

```text
Docling
   OR
PaddleOCR-VL
```

不同时维护多套主要解析系统。

统一接口：

```python
class DocumentParser:
    def parse(self, source) -> StructuredDocument:
        ...
```

底层实现可以是：

```text
DoclingParser
```

或者：

```text
PaddleOCRVLParser
```

上层逻辑不关心具体实现。

这样未来可以替换 parser，而不影响：

```text
Normalization
Chunking
Embedding
Vector Store
```

---

## 4. 输入处理策略

```text
Input
│
├── PDF / Image / Document
│      ↓
│   Unified Document Parser
│      ↓
│   StructuredDocument
│
└── Plain Text
       ↓
   Text Structure Detection
       ↓
   Structured / Unstructured
```

对于 PDF：

- 有 Native Text
- 扫描 PDF
- 混合 PDF

尽量交给选定的 Document Parser 自己统一处理。

项目本身不开发复杂的：

```text
OCR
 ↓
Layout Detection
 ↓
Fusion
 ↓
Structure Reconstruction
```

Pipeline。

---

## 5. StructuredDocument

Parser 输出统一的数据结构，例如：

```text
StructuredDocument
├── metadata
├── pages
└── blocks
    ├── Heading
    ├── Paragraph
    ├── List
    ├── Table
    ├── Figure
    ├── Formula
    └── Code
```

每个 Block 尽量包含：

```text
text
type
page
bbox
heading_path
parent
metadata
```

这一层是 Parser 和 Chunker 之间的统一边界。

---

## 6. Normalization / Cleaning

解析完成后先执行确定性清理：

```text
StructuredDocument
       ↓
Normalizer
       ↓
CleanStructuredDocument
```

主要职责：

- 删除 Header / Footer / Page Number
- 删除重复文本
- 清理空白和异常字符
- 合并跨页 Paragraph
- 修复明显 Reading Order 问题
- 合并破碎 Block
- 统一 Metadata
- 保留特殊 Block

尽量使用规则代码，不使用 LLM。

接口：

```python
class DocumentNormalizer:
    def normalize(
        self,
        document: StructuredDocument
    ) -> StructuredDocument:
        ...
```

---

## 7. Chunking 总体策略

Chunking 使用分层策略：

```text
Structured Content
        ↓
Structure-aware Split
        ↓
Size Check
   ┌───────┼────────┐
   │       │        │
 Too Big  Good    Too Small
   │                │
Semantic Split     Merge
   │                │
   └────────┬───────┘
            ↓
       Final Chunks
```

---

## 8. 第一层：规则结构分割

只要存在可靠结构，就优先使用代码规则。

规则处理：

- Heading
- Section
- Paragraph
- List
- Table
- Code Block
- Formula
- Figure / Caption

典型逻辑：

```text
Heading
  ↓
收集属于该 Heading 的 Blocks
  ↓
形成逻辑 Section
```

接口：

```python
class StructureChunker:
    def split(
        self,
        document: StructuredDocument
    ) -> list[ChunkCandidate]:
        ...
```

---

## 9. 第二层：Size Control

每个候选 Chunk 做长度判断：

```text
ChunkCandidate
      ↓
Token Count
      ↓
┌────────────┬─────────────┬─────────────┐
│            │             │
Too Large   Suitable      Too Small
│            │             │
Split        Keep          Merge
```

配置：

```text
max_tokens
target_tokens
min_tokens
overlap
```

---

## 10. 第三层：Semantic Segmentation

只有以下情况才调用：

- 长 Section 没有进一步结构
- 连续纯文本明显包含多个主题
- Rule-based split 无法得到合理 Chunk

默认不让所有内容经过模型。

### 可选方案 A：Embedding-based

```text
Long Text
   ↓
Sentence Split
   ↓
Embedding
   ↓
Similarity Change
   ↓
Semantic Boundaries
```

### 可选方案 B：LLM-based

项目规划中可以重点考虑：

```text
Large Unstructured Segment
          ↓
         LLM
          ↓
Semantic Boundaries
          ↓
Sub-segments
```

LLM 定位：

> **fallback / conditional semantic segmenter**

而不是默认 Chunker。

统一接口：

```python
class SemanticSegmenter:
    def split(self, text: str) -> list[str]:
        ...
```

底层以后可以替换：

```text
EmbeddingSemanticSegmenter
LLMSemanticSegmenter
```

---

## 11. Small Chunk Merge

过小 Chunk 不直接进入 Embedding。

可以按照规则：

```text
Small Chunk
    ↓
是否有 Parent Heading？
    ↓
与 Parent Context / Neighbor 合并
```

优先考虑：

1. 同一 Section 内相邻 Chunk
2. 相同 Heading Path
3. 不跨越明显 Topic Boundary

接口：

```python
class ChunkMerger:
    def merge(
        self,
        chunks: list[ChunkCandidate]
    ) -> list[Chunk]:
        ...
```

---

## 12. 特殊 Block 策略

### Table

- 尽量保持完整
- 保留 Caption / Header
- 超长表格按 Row group 拆分
- 每个 Chunk 继承 Table Header

### Code

- 保持 Code Block
- 不按普通句子规则切
- 超长时使用 code-aware split

### Formula

- 保持公式完整
- 与解释它的 Paragraph 保持上下文关系

### Figure

基础版本：

- 保留 Caption
- 如果 Parser / VLM 可以生成描述，则保存 description
- 以后再考虑 Multimodal Retrieval

---

## 13. Final Chunk Schema

最终进入 Embedding 的 Chunk 建议包含：

```text
Chunk
├── id
├── document_id
├── text
├── chunk_type
├── heading_path
├── page_start
├── page_end
├── source
├── language
└── metadata
```

必要时还可以保存：

```text
parent_chunk_id
bbox
parser_version
chunker_version
```

---

## 14. Embedding 层

统一接口：

```python
class Embedder:
    def embed(
        self,
        texts: list[str]
    ) -> list[Vector]:
        ...
```

流程：

```text
Final Chunks
     ↓
Embedder
     ↓
Vectors
```

Chunking 和 Embedding 解耦。

---

## 15. Vector Store

统一接口：

```python
class VectorStore:
    def upsert(
        self,
        chunks: list[Chunk],
        vectors: list[Vector]
    ):
        ...
```

保存：

```text
Chunk Text
+
Embedding
+
Metadata
```

具体实现以后可以是：

```text
PostgreSQL + pgvector
Qdrant
Milvus
...
```

但不影响 ingestion 主流程。

---

## 16. Pipeline 类

整体可以有：

```python
class IngestionPipeline:

    def ingest(self, source):

        document = parser.parse(source)

        document = normalizer.normalize(document)

        candidates = structure_chunker.split(document)

        chunks = hierarchical_chunker.process(candidates)

        vectors = embedder.embed(
            [chunk.text for chunk in chunks]
        )

        vector_store.upsert(
            chunks,
            vectors
        )
```

其中：

```text
hierarchical_chunker
```

内部负责：

```text
Size Check
+
Semantic Split
+
Small Chunk Merge
```

---

## 17. 推荐模块结构

```text
rag/
│
├── parsing/
│   ├── base.py
│   ├── docling_parser.py
│   └── paddleocrvl_parser.py
│
├── normalization/
│   └── document_normalizer.py
│
├── chunking/
│   ├── structure_chunker.py
│   ├── semantic_segmenter.py
│   ├── chunk_merger.py
│   └── hierarchical_chunker.py
│
├── embedding/
│   └── embedder.py
│
├── storage/
│   └── vector_store.py
│
├── models/
│   ├── structured_document.py
│   └── chunk.py
│
└── ingestion_pipeline.py
```

项目初期不一定需要真的拆这么多文件，但职责可以按照这个思路设计。

---

## 18. 第一版实际技术策略

建议第一版保持简单：

```text
Document Parser
    ↓
Docling OR PaddleOCR-VL

Normalization
    ↓
Rule-based Python Code

Structure Split
    ↓
Rule-based Python Code

Too Large?
    ↓
No → Keep
Yes → Semantic Segmentation

Semantic Segmentation
    ↓
优先测试 LLM

Too Small?
    ↓
Rule-based Merge

Final Chunks
    ↓
Embedding
    ↓
Vector Store
```

---

## 19. 核心设计原则

### Parser 单一化

项目中选择一个主 Document Parser 即可：

```text
Docling
OR
PaddleOCR-VL
```

避免维护多个并行 Pipeline。

---

### 规则优先

能通过：

```text
Heading
Paragraph
Token Count
Block Type
```

解决的问题，直接使用代码。

---

### LLM 兜底

LLM 只负责：

> 无法通过已有结构和确定性规则正确判断的语义边界。

---

### Chunking 分层化

不是：

```text
选择一种 Chunking Algorithm
```

而是：

```text
Structure
   ↓
Size
   ↓
Semantic
   ↓
Merge
```

---

### 组件解耦

```text
Parser
 ↓
Normalizer
 ↓
Chunker
 ↓
Embedder
 ↓
VectorStore
```

每层有统一接口。

以后替换任何一个实现时，不应该影响其他层。

---

## 20. 最终规划图

```text
                         Input
                           │
                 Document / Plain Text
                           │
                           ▼
                  Unified Parser Layer
                           │
               Docling OR PaddleOCR-VL
                           │
                           ▼
                 Structured Document
                           │
                           ▼
                Normalization / Cleaning
                           │
                           ▼
                Rule-based Structure Split
                           │
                           ▼
                      Size Check
                    /      |       \
                   /       |        \
             Too Large   Good    Too Small
                 │         │          │
                 ▼         │          ▼
          Semantic Split   │     Rule-based Merge
          LLM / Embedding  │          │
                 │         │          │
                 └─────────┼──────────┘
                           ▼
                      Final Chunks
                           │
                           ▼
                        Embedding
                           │
                           ▼
                      Vector Store
```

这套 Pipeline 的核心是：

> **Parser 单一化、规则优先、LLM 条件式参与、Chunking 分层化、组件之间保持解耦。**
