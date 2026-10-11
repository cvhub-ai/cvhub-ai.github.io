# Document Processing

## 1. 文档处理的作用

在建立索引之前，需要先把原始文档转换成稳定、可检索的文本表示。

一个简化流程是：

```
Raw Document / Bytes
        ↓
Format Detection & Parsing
        ↓
Character Decoding
        ↓
Language Detection
        ↓
Text Normalization
        ↓
Tokenization
        ↓
Optional Token Processing
        ↓
Processed Tokens
```

对于 PDF、DOCX 等复杂格式，通常先通过 Parser 提取文本和结构，而不是直接把整个文件按 UTF-8 解码。

## 2. ASCII、Unicode 与 UTF-8

### ASCII

ASCII 使用 7 bit，可以表示 128 个字符，包括：

- 英文字母
- 数字
- 常见标点
- 控制字符

编码范围：

```text
0x00 - 0x7F
```

它无法覆盖中文、日文等大量字符。

### Unicode

Unicode 是统一字符标准。它给每个字符分配唯一的 Code Point，例如：

```text
A  -> U+0041
中 -> U+4E2D
```

Unicode 本身不是具体的字节编码方式。

### UTF-8

UTF-8 是 Unicode 的一种编码方式。

特点：

1. 兼容 ASCII
2. 变长编码
3. 具有自同步能力

编码模式：

```text
1 byte:  0xxxxxxx
2 bytes: 110xxxxx 10xxxxxx
3 bytes: 1110xxxx 10xxxxxx 10xxxxxx
4 bytes: 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```

常见情况：

- ASCII 字符：1 byte
- 很多欧洲字符：2 bytes
- 常见中日韩字符：3 bytes
- 很多 Emoji：4 bytes

## 3. Tokenization

Tokenization 的目标是把文本拆分成适合后续索引和检索的 Token。

例如：

```text
Information retrieval is useful.
```

可能得到：

```text
["information", "retrieval", "is", "useful"]
```

不同语言需要不同策略。英文可以较多依赖空格和标点，但中文、日文通常需要专门的分词方法。

## 4. Text Normalization

Normalization 用于减少表面形式差异。

常见操作包括：

- Lowercasing
- Unicode normalization
- 标点规范化
- 特殊字符处理
- 数字和单位规范化

是否删除 Stopwords，需要根据具体任务决定。

## 5. Stemming 与 Lemmatization

二者通常发生在 Tokenization 之后。

### Stemming

Stemming 通常使用启发式规则去除词缀。

特点：

- 通常不依赖完整词典
- 速度快
- 输出结果不一定是真实单词

例如：

```text
studies -> studi
```

### Lemmatization

Lemmatization 根据形态学信息将词还原到词元形式。

例如：

```text
studies -> study
running -> run
better   -> good
```

特点：

- 通常依赖词典、规则或语言模型
- 结果更接近真实词元
- 计算成本通常高于简单 Stemming

## 6. Document Classification

Document Classification 是把文档分配到一个或多个类别。

常见方法有两类：

### Rule-based

使用启发式规则，例如：

- 具有区分能力的词频特征
- 特定关键词
- 文档长度和结构
- 文件类型
- 作者、来源等元数据
- 语言信息

### Model-based

使用训练得到的分类器，例如：

- Logistic Regression
- SVM
- Neural Network
- Transformer-based Classifier

### 常见难点

- 语言歧义
- 多语言处理
- 脏数据和不完整文档
- 大规模可扩展性
- Precision / Recall / F1 的平衡
- 文档分布和类别随时间变化
- Multi-label Classification
