# Query Processing 与 Phrase Retrieval

## 1. Query Processing

用户输入 Query 后，需要先执行与文档索引兼容的处理。

例如：

```text
Raw Query
   ↓
Normalization
   ↓
Tokenization
   ↓
Optional Stemming / Lemmatization
   ↓
Terms
```

Query Processing 使用的规则应尽量与文档索引阶段一致。

## 2. AND 查询优化

假设：

```text
A AND B AND C
```

并且：

```text
df(A) = 100000
df(B) = 200
df(C) = 20
```

通常优先执行：

```text
C AND B
```

再与：

```text
A
```

求交集。

原因是 DF 较小的 Posting List 可以更快缩小中间结果集。

因此经典启发式规则是：

> 对 AND 查询，优先处理 DF 较小的 Posting List。

## 3. NOT 查询

例如：

```text
apple AND banana AND NOT orange
```

通常先得到：

```text
apple AND banana
```

的候选结果，再排除包含 `orange` 的文档。

即：

```text
(A ∩ B) \ C
```

通常不建议先构造完整的 `NOT orange` 补集，因为补集可能非常大。

## 4. Phrase Retrieval

普通倒排索引只知道：

```text
machine 和 learning 是否出现在同一个文档
```

但无法直接知道：

```text
"machine learning"
```

是否作为连续短语出现。

因此需要额外的索引设计。

## 5. Biword Index

Biword Index 将相邻的两个 Token 组合成一个索引项。

例如：

```text
machine learning model
```

转换成：

```text
machine learning
learning model
```

查询：

```text
"machine learning model"
```

可以拆成：

```text
"machine learning"
AND
"learning model"
```

优点：

- 查询简单
- 两词短语非常直接

缺点：

- Dictionary 规模明显增加
- 多词短语可能产生 False Positive
- 不适合灵活的距离查询

不需要分别构建所有三词、四词、五词索引。

## 6. Positional Index

Positional Index 在 Posting List 中记录词项的位置。

例如：

```text
machine:
D1 -> [2, 8]
D2 -> [5]

learning:
D1 -> [3, 10]
D2 -> [6]
```

查询：

```text
"machine learning"
```

要求：

```text
position(learning)
=
position(machine) + 1
```

因此：

```text
D1: 2 -> 3   match
D2: 5 -> 6   match
```

## 7. Positional Index 的优势

它可以支持：

- Phrase Search
- Proximity Search
- 任意长度短语
- NEAR / k 类查询

例如：

```text
machine NEAR/5 learning
```

可以通过位置差计算完成。

代价是：

- 索引体积增大
- Posting List 更复杂
- 查询阶段需要额外的位置匹配
