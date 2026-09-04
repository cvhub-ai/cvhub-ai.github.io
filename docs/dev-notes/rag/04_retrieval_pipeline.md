# RAG Retrieval Pipeline

## 1. Retrieval 的目标

Retrieval 的任务不是简单地：

> 找到与 Query 向量最相似的几个 Chunk。

真正目标是：

> 从 Knowledge Base 中找到能够支持当前问题的高相关、高质量知识。

完整 Pipeline 可以表示为：

```text
User / Agent Query
       ↓
Query Processing
       ↓
 ┌─────┴─────┐
 ↓           ↓
Dense       Sparse
Retriever   Retriever
 ↓           ↓
Vector      BM25
 └─────┬─────┘
       ↓
     Fusion
       ↓
Candidate Results
       ↓
    Reranker
       ↓
Final Top-K
       ↓
Context Builder
```

---

## 2. Query Processing

原始 Query 不一定是最适合检索的 Query。

因此可以增加：

```text
Original Query
      ↓
Query Processor
      ↓
Retrieval Query
```

但 Query Processing 应按需求使用，而不是默认堆叠所有方法。

---

## 3. Query Normalization

基础处理包括：

- Whitespace Cleaning
- Unicode Normalization
- Case Handling
- Language Detection
- Typo Handling
- Domain-specific normalization

目标：

> 减少没有语义意义的格式差异。

对于产品编号、错误代码等内容要谨慎，不能因为 normalization 破坏精确信息。

---

## 4. Query Rewrite

将用户表达转换为更适合 Retrieval 的表达。

例如：

```text
User:
"那个 OCR 服务最大能传多大的图？"

Rewrite:
"OCR service maximum supported image size input limit"
```

目的：

- 消除口语表达
- 补充上下文
- 明确实体

风险：

> Rewrite 可能改变用户真实意图。

因此原始 Query 应保留用于 Debugging / Evaluation。

---

## 5. Query Expansion

在原 Query 基础上增加：

- Synonym
- Abbreviation
- Related Term
- Domain Terminology

例如：

```text
OCR
→ Optical Character Recognition
```

可以提高 Recall。

---

## 6. Multi-query Retrieval

一个问题生成多个 Retrieval Query：

```text
Original Query
      ↓
 ┌────┼────┐
 Q1   Q2   Q3
 ↓    ↓    ↓
Retrieval
 └────┬────┘
      ↓
    Fusion
```

适用于：

- 用户问题表达模糊
- 同一概念存在不同表达
- 希望提高 Recall

缺点：

- Retrieval Cost 增加
- Duplicate 增加
- Fusion 更复杂

---

## 7. Query Decomposition

复杂问题可以拆成多个子问题。

例如：

```text
Question
 ↓
Sub-query A
Sub-query B
Sub-query C
 ↓
Retrieve Separately
```

适合：

- Multi-hop Question
- 多条件问题
- 需要组合多个知识来源的问题

---

## 8. HyDE

HyDE：

**Hypothetical Document Embeddings**

基本思路：

```text
Query
 ↓
LLM generates hypothetical answer/document
 ↓
Embedding
 ↓
Vector Retrieval
```

目的：

> 使用更接近文档表达方式的文本进行向量检索。

它属于高级 Query Transformation，不应该默认用于所有 Query。

---

## 9. Metadata Filter Extraction

Query 中可能包含明确条件：

```text
"查 2026 年版本的 OCR 文档"
```

可以提取：

```text
year = 2026
service = OCR
```

然后：

```text
Semantic Search
+
Metadata Filter
```

这比单纯依赖 Embedding 更可靠。

---

# 10. Dense Retrieval

Dense Retrieval：

```text
Query
 ↓
Embedding Model
 ↓
Query Vector
 ↓
Vector Search
 ↓
Top-K Chunks
```

优势：

- Semantic Similarity
- 同义表达
- Cross-lingual Retrieval
- 不要求关键词完全一致

弱点：

- 精确编号可能表现一般
- 相似语义不一定代表真正相关
- Domain Shift 会影响效果

---

# 11. Sparse Retrieval

典型方法：

```text
BM25
```

它依赖文本中的实际 Term。

优势：

- Product ID
- Error Code
- Model Name
- API Name
- Exact Keyword
- Domain Terminology

---

## 12. BM25 基础

BM25 的核心来自：

```text
TF
+
IDF
+
Document Length Normalization
```

### TF

Term Frequency：

> 一个词在当前文档中出现多少次。

### IDF

Inverse Document Frequency：

> 一个词在整个语料中越少见，通常区分能力越强。

### Document Length

BM25 会避免长文档仅仅因为包含更多词而获得不合理优势。

不需要为了使用 BM25 自己实现公式，但需要理解：

> BM25 是 lexical matching，不是 semantic embedding search。

---

## 13. Tokenizer 对 Sparse Retrieval 的影响

不同语言：

```text
English
German
Chinese
```

Tokenization 不同。

还需要考虑：

- Compound Words
- Stemming
- Lemmatization
- Stop Words
- Special Characters
- Product Codes

因此 BM25 效果也依赖文本分析 Pipeline。

---

# 14. Metadata Retrieval / Filtering

有些条件不应该通过 Semantic Similarity 判断。

例如：

```text
language = de
version = 3
service = OCR
document_type = API_DOC
```

可以直接使用 Metadata Filter。

典型方式：

```text
Filter
+
Vector Search
```

Filter 可以：

### Pre-filter

先过滤：

```text
All Chunks
 ↓
Metadata Filter
 ↓
Candidate Subset
 ↓
Vector Search
```

### Post-filter

先搜索再过滤。

通常需要根据数据库能力、数据规模和过滤条件选择。

---

# 15. Hybrid Retrieval

Dense 与 Sparse 各有优势。

因此：

```text
             Query
               │
       ┌───────┴───────┐
       ↓               ↓
Dense Retriever   Sparse Retriever
       ↓               ↓
Vector Top-K        BM25 Top-K
       └───────┬───────┘
               ↓
             Fusion
```

目标：

```text
Semantic Recall
+
Exact Keyword Recall
```

---

# 16. Fusion

多个 Retriever 返回不同：

```text
Document
Rank
Score Scale
```

因此不能简单假设：

```text
Vector Score 0.8
=
BM25 Score 0.8
```

两种 Score 的含义和范围可能完全不同。

需要 Fusion Strategy。

---

## 17. Reciprocal Rank Fusion

RRF：

**Reciprocal Rank Fusion**

核心思想：

> 更关注 Result 在各个 Retriever 中的排名，而不是直接比较原始 Score。

概念上：

```text
Vector Ranking
1 A
2 B
3 C

BM25 Ranking
1 B
2 D
3 A

       ↓
      RRF
       ↓
Combined Ranking
```

优势：

- 不要求不同 Retriever 的 Score 同尺度
- 简单
- 稳定
- 很适合 Hybrid Retrieval

---

## 18. Weighted Score Fusion

另一种方法：

```text
Final Score
=
α × Dense Score
+
β × Sparse Score
```

但前提通常需要：

```text
Score Normalization
```

否则两个不同尺度的 Score 无法合理直接相加。

优点：

- 可以显式控制权重

缺点：

- 权重需要 Evaluation
- Score Calibration 更复杂

---

# 19. Candidate Size

Retriever 通常不会只返回最终需要的数量。

例如：

```text
Dense Top 20
+
BM25 Top 20
 ↓
Fusion
 ↓
Top 20
 ↓
Reranker
 ↓
Final Top 5
```

因此需要区分：

```text
retrieval_top_k
rerank_top_n
final_top_k
```

这些参数应通过 Evaluation 调整。

---

# 20. Reranking

Retriever 的主要目标通常偏向：

> Recall。

Reranker 的目标：

> 对较小候选集进行更精确的相关性判断。

```text
Retrieve Top-N
      ↓
Reranker
      ↓
Final Top-K
```

---

## 21. Cross-Encoder Reranker

典型输入：

```text
(Query, Chunk)
 ↓
Cross Encoder
 ↓
Relevance Score
```

与 Embedding Retrieval 不同：

Dense Retrieval：

```text
Query → Vector
Chunk → Vector
Compare
```

Cross Encoder：

```text
Query + Chunk
      ↓
Model Jointly Reads Both
      ↓
Score
```

通常：

- 更准确
- 更慢

所以适合候选集较小时使用。

---

## 22. LLM Reranker

可以让 LLM 判断：

```text
Query
+
Candidate Documents
 ↓
Relevance Judgment
```

优点：

- 复杂语义理解强

缺点：

- 成本高
- 延迟高
- 输出稳定性需要控制

通常不是基础 RAG 的第一选择。

---

## 23. Rule-based Reranking

某些领域规则非常可靠。

例如：

```text
Exact Model Number Match
Unit Match
Language Match
Document Version
Service Name
```

可以作为：

- Filter
- Score Boost
- Reranking Rule

原则：

> 确定性条件不必全部交给模型判断。

---

# 24. Retrieval Score 与 Reranking Score

不要认为：

```text
Embedding Similarity Score
=
Reranking Relevance Score
```

它们来自不同模型，含义不同。

建议结果中保留：

```text
dense_score
sparse_score
fusion_rank
rerank_score
```

方便：

- Debugging
- Evaluation
- Observability

---

# 25. Parent-Child Retrieval

如果小 Chunk 检索准确但上下文不足：

```text
Small Child Chunk
 ↓
Retrieve
 ↓
Find Parent Section
 ↓
Return Parent Context
```

即：

```text
Small-to-Big Retrieval
```

用于平衡：

```text
Precision
vs
Context Completeness
```

---

# 26. Multi-stage Retrieval

更复杂系统可以：

```text
Large Knowledge Base
      ↓
Stage 1: Broad Retrieval
      ↓
Stage 2: Filter / Fusion
      ↓
Stage 3: Rerank
      ↓
Final Context
```

不要一开始过度设计。

基础系统通常：

```text
Hybrid Retrieval
+
RRF
+
Cross-Encoder Reranker
```

已经是非常完整的 Retrieval Pipeline。

---

# 27. Retrieval Evaluation

Retriever 应使用真实 Query Dataset 测试。

常见指标：

```text
Recall@K
Precision@K
Hit Rate
MRR
nDCG
```

尤其需要关注：

> 正确知识是否进入候选集合。

如果正确 Chunk 根本没有被 Retriever 找到，后面的 Reranker 和 LLM 通常无法补救。

---

# 28. 推荐认知框架

```text
                     Query
                       ↓
                Query Processing
                       ↓
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
      Dense          Sparse        Metadata
    Retrieval       Retrieval       Filter
        ↓              ↓              ↓
        └──────────────┬──────────────┘
                       ↓
                     Fusion
                       ↓
                  Candidate Set
                       ↓
                    Reranker
                       ↓
                   Final Top-K
                       ↓
                 Context Builder
```

核心原则：

> **Retriever 首先保证 Recall，Fusion 组合不同 Retrieval 信号，Reranker 再提高 Precision。**
