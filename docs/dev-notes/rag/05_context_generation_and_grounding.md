# RAG Context Building、Generation 与 Grounding

## 1. Context Building 的位置

Retrieval 并不是 RAG Query Pipeline 的终点。

```text
Query
 ↓
Retrieval
 ↓
Fusion
 ↓
Reranking
 ↓
Retrieved Chunks
 ↓
Context Building
 ↓
LLM / Agent
```

Context Builder 的职责：

> 将检索到的知识转换成适合后续 LLM / Agent 使用的上下文。

不能简单理解为：

```text
Top-K Chunks
 ↓
全部拼接
 ↓
LLM
```

---

## 2. 为什么需要 Context Builder

Retrieval Results 可能存在：

- Duplicate Chunks
- Overlapping Chunks
- 相邻 Chunk
- 不同 Document
- 不同 Score
- 不同 Parent Section
- 超过 Context Window
- 顺序混乱

因此需要：

```text
Retrieved Results
       ↓
Deduplicate
       ↓
Merge / Expand
       ↓
Order
       ↓
Budget
       ↓
Format
       ↓
Final Context
```

---

# 3. Deduplication

Chunk Overlap、Hybrid Retrieval 和 Multi-query 都可能产生重复结果。

例如：

```text
Chunk A
Chunk A'
Chunk A''
```

内容高度重叠。

如果全部进入 Context：

- 浪费 Token
- 放大某个 Source 权重
- 降低 Context Diversity

因此需要：

```text
Exact Deduplication
Near-Duplicate Detection
Overlap Deduplication
```

---

# 4. Adjacent Chunk Merge

可能检索到：

```text
Section 3
├── Chunk 10
├── Chunk 11 ← retrieved
├── Chunk 12 ← retrieved
└── Chunk 13
```

如果 11 和 12 连续，可以考虑：

```text
Chunk 11 + Chunk 12
```

恢复更完整上下文。

但不能无限 Merge，否则重新变成巨大 Chunk。

---

# 5. Parent Context Expansion

如果检索的是 Child Chunk：

```text
Heading
  ↓
Paragraph
  ↓
Small Chunk ← matched
```

可以补充：

- Parent Heading
- Section Title
- Nearby Paragraph
- Parent Chunk

这样：

```text
Retrieval Unit
≠
Final Context Unit
```

这是很重要的设计思想。

---

# 6. Context Ordering

最终 Context 的顺序会影响 LLM。

可以考虑：

- Relevance Order
- Document Order
- Source Grouping
- Chronological Order
- Logical Structure

例如同一文档的多个 Chunk，可以恢复原文顺序。

---

# 7. Token Budget

LLM Context Window 有限制。

假设：

```text
Model Context = 32k
```

不能全部用于 Retrieved Context。

还需要：

```text
System Prompt
User Query
Conversation History
Tool Results
Output Tokens
```

因此需要 Context Budget：

```text
Available Context Tokens
      ↓
Select / Trim Retrieved Content
```

---

# 8. Context Compression

如果 Retrieved Chunk 太长，可以压缩。

方法包括：

### Extractive Compression

只保留与 Query 最相关的句子 / 段落。

### Abstractive Compression

使用模型总结 Retrieved Content。

风险：

> Abstractive Compression 本身可能引入信息变化或幻觉。

因此需要根据场景谨慎使用。

---

# 9. Diversity

如果 Top-K 全来自一个非常相似的 Section：

```text
Chunk A1
Chunk A2
Chunk A3
Chunk A4
Chunk A5
```

可能遗漏其他重要 Source。

可以考虑：

```text
Diversity-aware Selection
```

目标是在：

```text
Relevance
+
Coverage
```

之间取得平衡。

---

# 10. Lost in the Middle

LLM 对长 Context 中不同位置的信息利用能力可能不完全一致。

因此 Context Builder 不应只关注：

```text
Context 能不能塞进去
```

还应该关注：

```text
重要信息放在哪里
Context 是否过长
是否存在大量无关内容
```

高质量、精简 Context 往往比盲目增加 Top-K 更有效。

---

# 11. Context Format

Context 可以采用明确结构：

```text
[Source 1]
Document: ...
Page: ...
Section: ...
Content:
...

[Source 2]
...
```

好处：

- LLM 更容易区分 Source
- Citation Mapping 更简单
- Debugging 更清楚

---

# 12. Source Tracking

从 Ingestion 开始就应该保持：

```text
Chunk
 ↓
Document
 ↓
Source
```

例如：

```text
chunk_id
document_id
document_version
page
section
heading_path
bbox
source_uri
```

否则到了 Generation 阶段很难可靠生成 Citation。

---

# 13. Citation

Citation 的目标：

> 让最终回答能够追踪到实际 Knowledge Source。

例如：

```text
Answer Claim
 ↓
Retrieved Chunk
 ↓
Document
 ↓
Page / Section
```

Citation 不应该只记录文件名。

更完整可以包含：

```text
document_id
document_title
page
section
bbox
source
version
```

---

# 14. PDF Citation

对于 PDF：

```text
Document
 ↓
Page
 ↓
Block / BBox
```

如果 Parser 保留 Bounding Box，可以进一步支持：

> 在原 PDF 页面中定位对应证据。

这也是为什么 Parsing 阶段保留结构 Metadata 很重要。

---

# 15. 跨页 Chunk Citation

如果一个 Chunk 来自：

```text
Page 10
+
Page 11
```

Metadata 不应该强制只有一个 page。

可以表示：

```text
pages = [10, 11]
```

或：

```text
page_start
page_end
```

---

# 16. Table Citation

Table 的 Citation 应尽量能够追踪：

```text
Document
Page
Table
Row / Column
```

至少应保留 Table Block 身份，而不是完全丢失结构后只引用一段文本。

---

# 17. Grounding

Grounding：

> 最终输出应尽量建立在 Retrieved Evidence 上。

简单理解：

```text
Question
+
Retrieved Evidence
 ↓
Answer
```

而不是：

```text
Question
 ↓
LLM Free Generation
```

---

# 18. Faithfulness

Faithfulness 关注：

> 回答中的 Claim 是否能够被 Context 支持。

一个回答可能：

- 语言很好
- 看起来合理
- 甚至事实正确

但如果它不是根据当前 Context 得出的，在严格 RAG Evaluation 中仍可能被视为 Grounding 不足。

---

# 19. Answer Relevance 与 Faithfulness

两者不同。

### Answer Relevance

回答是否真正解决用户问题。

### Faithfulness

回答是否受到 Retrieved Evidence 支持。

理想状态：

```text
Relevant
+
Faithful
```

---

# 20. Prompt Construction

传统 RAG：

```text
System Instruction
+
Retrieved Context
+
User Query
 ↓
Prompt
 ↓
LLM
```

Prompt 应明确告诉模型：

- Context 是参考资料
- 优先依据 Context
- 不知道时不要凭空补充
- Citation 的输出规则

---

# 21. Retrieved Content 不是 Instruction

这是重要安全原则：

```text
Retrieved Document
=
Untrusted Data
```

而不是：

```text
System Instruction
```

如果文档中出现：

```text
Ignore all previous instructions...
```

LLM 不应该把它当成系统指令。

因此 Prompt / Agent 设计需要明确区分：

```text
Trusted Instructions
vs
Retrieved Content
```

---

# 22. Traditional RAG

传统问答系统可以：

```text
Query
 ↓
RAG Pipeline
 ↓
Context
 ↓
Prompt Builder
 ↓
LLM
 ↓
Answer
```

这里 Generator 可以属于 RAG 服务本身。

---

# 23. Agentic Architecture

在 Agent 系统中可以：

```text
User
 ↓
Agent
 ↓
RAG Retrieval
 ↓
Context
 ↓
Agent Reasoning
 ↓
LLM / Tools
 ↓
Answer
```

此时：

```text
RAG
=
Knowledge Retrieval Capability
```

而：

```text
Agent
=
Reasoning / Orchestration
```

两种设计都合理，取决于系统职责边界。

---

# 24. RAG 与 Tool Result

Agent 场景中最终 Context 可能来自：

```text
RAG Knowledge
+
Tool Result
+
Conversation Context
+
User Input
```

因此 Agent 最终负责组合多个信息源。

RAG 不应该把：

```text
Knowledge Retrieval
```

和：

```text
External Action Execution
```

混成同一职责。

---

# 25. Context Builder 输出

一个结构化输出可以是：

```text
ContextResult
├── query
├── chunks
│   ├── text
│   ├── score
│   ├── source
│   └── metadata
├── citations
└── token_count
```

这样后续 Agent / Generator 不依赖某个特定 Vector DB。

---

# 26. Context Quality

Context Builder 应追求：

```text
Relevant
Complete
Non-duplicate
Traceable
Within Token Budget
Properly Ordered
```

而不是：

```text
More Chunks = Better
```

---

# 27. Context Evaluation

可以关注：

```text
Context Precision
Context Recall
```

### Context Precision

Retrieved Context 中有多少内容真正相关。

### Context Recall

回答问题所需的证据有多少被 Retrieved Context 覆盖。

这两者反映：

```text
Too Much Noise
vs
Missing Evidence
```

---

# 28. Citation Evaluation

Citation 需要评估：

```text
Citation Correctness
Citation Completeness
Citation Source Accuracy
```

不能仅仅检查“有没有 Citation”。

---

# 29. 最终框架

```text
Retrieved Candidates
        ↓
     Reranker
        ↓
   Relevant Chunks
        ↓
   Context Builder
   ├── Deduplicate
   ├── Merge
   ├── Expand Parent
   ├── Order
   ├── Budget
   └── Format
        ↓
   Grounded Context
        ↓
   LLM / Agent
        ↓
      Answer
        ↓
     Citation
```

核心原则：

> **Retrieval 决定找到了什么，Context Building 决定真正给模型看什么，Grounding 决定最终回答是否可靠地建立在这些证据之上。**
