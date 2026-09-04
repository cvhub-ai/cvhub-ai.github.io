# RAG Embedding、Vector Store 与 Indexing

## 1. 整体位置

```text
Chunk
 ↓
Embedding Model
 ↓
Vector
 ↓
Vector Store
 ↓
Vector Index
 ↓
Retrieval
```

这一阶段解决两个核心问题：

1. 如何把知识转换成可比较的数学表示？
2. 如何高效地从大量知识中找到最相关的内容？

---

## 2. Embedding

Embedding：

> 将文本映射到高维向量空间。

例如：

```text
"汽车用高强度塑料"

↓

[0.13, -0.72, 0.44, ...]
```

语义相近的文本通常在向量空间中距离更近。

---

## 3. Document Embedding 与 Query Embedding

建立索引：

```text
Chunk
 ↓
Embedding Model
 ↓
Document Vector
```

查询：

```text
Query
 ↓
Same / Compatible Embedding Model
 ↓
Query Vector
```

然后：

```text
Query Vector
      ↓
Similarity Search
      ↓
Document Vectors
```

关键原则：

> Query Embedding 与 Document Embedding 必须位于兼容的向量空间。

因此不能随意使用不同 Embedding Model。

---

## 4. Embedding Model 的主要属性

选择模型时需要关注：

### Dimension

例如：

```text
384
768
1024
1536
...
```

Dimension 会影响：

- Storage
- Memory
- Search Cost
- Index Size

维度更高不自动代表效果更好。

### Max Sequence Length

决定单次能够处理的最大文本长度。

如果 Chunk 超过模型限制：

- 会被截断
- 或需要重新切分

### Language Support

模型可能是：

```text
English
Chinese
Multilingual
```

### Domain

通用模型不一定在专业领域表现最佳。

例如：

- Medical
- Legal
- Construction
- Code

都可能存在领域差异。

### Query / Document Instruction

部分 Embedding Model 对 Query 和 Document 使用不同 prefix / instruction。

例如概念上：

```text
query: ...
passage: ...
```

因此必须遵循具体模型的推荐使用方式。

---

## 5. Multilingual Embedding

现代 Multilingual Model 可以支持：

```text
中文 Query
 ↓
Embedding
 ↓
英文 / 德文 Document
```

即：

```text
Cross-lingual Retrieval
```

但：

> 支持某种语言不代表在该语言和特定领域上的效果一定足够好。

推荐流程：

```text
Multilingual Knowledge Base
        ↓
Multilingual Embedding Model
        ↓
Real Retrieval Evaluation
        ↓
效果不足
        ↓
考虑领域模型 / Fine-tuning
```

通常不建议项目一开始就 Fine-tune。

---

## 6. Similarity Metric

常见向量相似度 / 距离：

### Cosine Similarity

比较向量方向：

```text
cos(A, B)
```

常用于语义 Embedding。

### Dot Product

```text
A · B
```

如果两个向量都进行了 L2 Normalization：

```text
Dot Product
≈
Cosine Similarity
```

### Euclidean Distance / L2

比较空间距离：

```text
||A - B||
```

使用哪一种 Metric 应遵循 Embedding Model 和 Vector Store 的设计要求。

---

## 7. Embedding Normalization

常见：

```text
normalize_embeddings=True
```

将向量长度归一化。

好处之一是：

```text
Normalized Vector
+
Dot Product
```

可以直接表示 Cosine Similarity。

但是否应该 Normalize，需要根据具体模型建议决定。

---

## 8. Embedding Model Version

这是生产系统非常重要的 Metadata。

例如：

```text
embedding_model = bge-m3
embedding_version = xxx
dimension = 1024
```

如果从：

```text
Model A
```

切换到：

```text
Model B
```

旧 Vector 与新 Vector 通常不能直接混用。

常见迁移：

```text
New Embedding Model
       ↓
Re-embed Chunks
       ↓
Build New Index
       ↓
Evaluate
       ↓
Switch Index
```

---

## 9. Embedding 计算

对于大量文档：

```text
Chunks
 ↓
Batch Embedding
 ↓
Vectors
```

常见优化：

- Batch Processing
- GPU Inference
- Async Pipeline
- Embedding Cache
- Document Hash
- Incremental Embedding

如果 Chunk 内容没有变化，不应该无意义重复计算 Embedding。

---

## 10. Vector Store 保存什么

逻辑上需要维护：

```text
Vector Record
├── vector
├── chunk_id
└── metadata
```

有些系统同时保存：

```text
text
```

也有系统只保存：

```text
vector + chunk_id
```

然后通过 ID 到其他数据库读取正文。

因此更准确的定义是：

> Vector Store 保存用于 Retrieval 的 Vector，并维护 Vector 与原始 Chunk / Metadata 的关联。

---

## 11. Vector Database

Vector Database 是针对：

```text
Vector Storage
Vector Index
Nearest Neighbor Search
```

优化的数据库或存储系统。

常见专用系统：

- Qdrant
- Milvus
- Pinecone
- Weaviate

也可以使用：

```text
PostgreSQL
+
pgvector
```

因此：

```text
Vector Database
≠
必须使用一种独立数据库产品
```

---

## 12. Exact Search

最直接的方法：

```text
Query Vector
 ↓
Compare Vector 1
Compare Vector 2
Compare Vector 3
...
Compare Vector N
 ↓
Top-K
```

这属于：

```text
Brute Force / Exact Search
```

优点：

- Recall 精确
- 实现简单

缺点：

- 数据规模大时成本高

---

## 13. ANN

大规模向量搜索通常使用：

**Approximate Nearest Neighbor**

即：

> 不保证逐个检查全部 Vector，而是通过索引快速找到高概率最近邻。

核心 trade-off：

```text
Search Speed
vs
Recall
```

---

## 14. HNSW

HNSW：

**Hierarchical Navigable Small World**

核心思想：

> 将相似 Vector 建立多层 Graph，通过 Graph Navigation 快速接近 Query 的邻域。

简单理解：

```text
High Layer
节点少，快速定位
    ↓
Middle Layer
进一步缩小范围
    ↓
Bottom Layer
寻找局部近邻
```

关键词：

```text
Graph
Neighbor
Navigation
Hierarchy
```

常见参数：

```text
M
efConstruction
efSearch
```

通常：

```text
efSearch ↑
 ↓
搜索节点更多
 ↓
Recall ↑
Latency ↑
```

---

## 15. IVF

IVF：

**Inverted File Index**

核心思想：

> 先将 Vector 分区 / 聚类，Query 时只搜索最相关的部分 Cluster。

```text
All Vectors
 ↓
Clustering
 ↓
Cluster 1
Cluster 2
Cluster 3
...
```

Query：

```text
Query
 ↓
Find Nearest Centroids
 ↓
Select Relevant Clusters
 ↓
Search Inside Clusters
```

关键词：

```text
Cluster
Centroid
Partition
```

常见参数：

```text
nlist
nprobe
```

通常：

```text
nprobe ↑
 ↓
Search More Clusters
 ↓
Recall ↑
Latency ↑
```

---

## 16. HNSW vs IVF

| 项目 | HNSW | IVF |
|---|---|---|
| 核心思想 | Graph Navigation | Clustering / Partition |
| 数据组织 | Neighbor Graph | Cluster |
| 搜索方式 | 沿 Graph 找邻居 | 先选 Cluster，再搜索 |
| 关键概念 | Graph / Neighbor | Centroid / Cluster |
| 典型调节 | efSearch | nprobe |

一句话：

> **HNSW = 建图找邻居**

> **IVF = 先分区，再在少数分区中找邻居**

---

## 17. HNSW 与 Knowledge Graph

两者完全不同。

### HNSW Graph

```text
Vector A ─ Vector B
   │          │
Vector C ─ Vector D
```

边的意义：

> 为 Vector Search 提供快速导航。

### Knowledge Graph

例如：

```text
Product
   │ contains
   ↓
Component
   │ depends_on
   ↓
Library
```

边具有业务语义。

因此：

```text
HNSW
=
Vector Index Data Structure

Knowledge Graph
=
Business Knowledge Representation
```

---

## 18. Relational DB、Vector Store、Graph DB

### Relational Database

适合：

- Documents
- Users
- Permissions
- Categories
- Structured Metadata
- Exact Conditions
- Numeric Data

### Vector Store

适合：

- Embedding
- Semantic Search
- Similarity Retrieval

### Graph Database

适合：

- Entity Relationship
- Multi-hop Relation
- Path Query
- Knowledge Graph

不要根据项目规模机械增加数据库。

---

## 19. PostgreSQL + pgvector

对于很多普通 RAG：

```text
PostgreSQL
+
pgvector
```

已经可以同时管理：

```text
Document Metadata
Chunk Text
Structured Metadata
Embedding
Vector Index
```

优势：

- 架构简单
- SQL Filter 方便
- 事务能力成熟
- 不需要额外数据库

什么时候考虑专用 Vector DB，需要根据：

- Vector 数量
- Search QPS
- Latency
- Distributed Scaling
- Filtering
- Operational Requirements

实际 Evaluation 决定。

---

## 20. Index Lifecycle

索引不是建立一次后永久不变。

需要考虑：

```text
Document Insert
Document Update
Document Delete

Embedding Model Update
Chunking Strategy Update

Metadata Schema Update
```

可能触发：

```text
Incremental Index
Re-embedding
Re-index
Full Index Rebuild
```

---

## 21. 最终框架

```text
Chunk
 ↓
Embedding Model
 ↓
Vector
 ↓
Similarity Metric
 ↓
Vector Store
 ↓
Vector Index
 ├── Exact
 └── ANN
      ├── HNSW
      └── IVF
 ↓
Top-K Search
```

核心原则：

> **Embedding 决定知识如何表示，Index 决定如何快速寻找，但最终方案仍然必须通过真实 Retrieval Evaluation 选择。**
