# Boolean Retrieval 与 Inverted Index

## 1. Boolean Retrieval

Boolean Retrieval 根据词项是否存在于文档中进行检索。

典型查询：

```text
apple AND banana
apple OR banana
apple AND NOT orange
```

结果通常是二元的：

```text
match / not match
```

经典布尔检索本身不提供相关性评分。

## 2. Term-Document Matrix

一种最直观的表示方式是词-文档矩阵：

| Term / Document | D1 | D2 | D3 |
|---|---:|---:|---:|
| apple | 1 | 1 | 0 |
| banana | 1 | 0 | 1 |
| orange | 0 | 1 | 0 |

其中：

- 行：Term
- 列：Document
- 1：词出现
- 0：词未出现

问题在于真实文档集合中矩阵通常极其稀疏，大量位置都是 0。因此实际系统通常使用倒排索引。

## 3. Forward Index 与 Inverted Index

### Forward Index

```text
Document -> Terms

例如：

D1 -> apple, banana, machine
```

### Inverted Index

```text
Term -> Documents

例如：

apple  -> [1, 3, 5]
banana -> [2, 3, 5, 8]
```

倒排索引只保存实际存在的 Term-Document 关系，因此比完整稀疏矩阵更适合大规模检索。

## 4. 倒排索引的主要组成

### Dictionary / Lexicon

保存：

- Term
- Posting List 的位置
- DF 等统计信息

### Posting List

保存包含该 Term 的文档。最简单形式：

```text
apple -> [1, 3, 5]
```

更完整的 Posting 还可以保存：

```text
DocID
Term Frequency
Positions
Field Information
```

## 5. DF 与 TF

### Document Frequency

DF 表示包含某个 Term 的不同文档数量。

例如：

```text
apple -> [1, 3, 5]
```

则：

```text
df(apple) = 3
```

### Term Frequency

TF 表示某个 Term 在一个文档中出现的次数。

例如：

```text
tf(apple, D1) = 5
```

## 6. 倒排索引构建

简化流程：

```text
Token Extraction
      ↓
(Term, DocID) Pairs
      ↓
Sorting
      ↓
Grouping / Merging
      ↓
Dictionary + Posting Lists
```

通常会按 Term 和 DocID 排序，再把相同 Term 的记录合并成 Posting List。

## 7. Posting List 的存储

从概念上可以使用数组、动态数组或链表。

但现代 IR 系统更常采用：

- 有序连续存储
- Block-based Storage
- Delta Encoding
- Variable-byte Encoding
- Bit Packing

原因是连续内存通常：

- 缓存友好
- 遍历速度快
- 更适合压缩

增量更新通常通过新的索引 Segment 和后续 Merge 完成，而不是持续修改一条普通链表。

## 8. Intersect Algorithm

对于：

```text
A = [1, 3, 5, 7, 9]
B = [2, 3, 6, 7, 10]
```

求：

```text
A AND B
```

使用双指针：

1. 两个指针分别指向两个 Posting List 当前元素
2. DocID 相等：加入结果，两个指针都前进
3. 不相等：较小 DocID 的指针前进
4. 任一 Posting List 结束时停止

结果：

```text
[3, 7]
```

两个 Posting List 长度分别为 `m` 和 `n` 时，基本算法复杂度为：

```text
O(m + n)
```

## 9. Skip Pointers

Skip Pointer 允许在 Posting List 中跳过一段数据，减少不必要的比较。

基本原则：**如果当前 DocID 小于目标 DocID，可以检查跳跃目标是否仍然不超过目标 DocID。如果安全，就执行跳跃。**

优点：

- 减少比较次数
- 加快部分长 Posting List 的交集运算

缺点：

- 增加额外存储
- 增加索引维护复杂度
- Skip 间隔需要权衡

经典教材常使用大约 `√N` 规模的跳跃设计作为启发式方案。这不代表查询复杂度会自动变为 `O(√N)`。
