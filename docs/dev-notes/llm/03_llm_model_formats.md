# 本地 LLM：模型文件与存储

> 本文讨论本地 LLM 的模型文件与存储，重点区分模型、量化方式与模型文件格式之间的关系，并介绍 GGUF 和 safetensors。

## 1. 什么是模型文件格式

训练后的模型权重最终需要保存到文件，这属于：

**Serialization / Model File Format**

文件格式解决的问题是：

> **模型权重、Metadata 等信息如何被保存和加载。**

常见格式包括：

- GGUF
- safetensors

模型文件格式与模型本身、数据精度以及量化方法不是同一个概念。

例如：

```text
Qwen3-8B       → Model
AWQ            → Quantization Method
Q4_K_M         → Quantization Type
GGUF           → File Format
safetensors    → File Format
```

## 2. GGUF

GGUF 是 llama.cpp 生态中非常重要的模型文件格式。

它可以包含：

- 模型权重
- 模型架构信息
- Tokenizer 信息
- Metadata
- 量化权重

> **GGUF 是文件格式，不等于量化算法。**

例如：

```text
Qwen-9B-Q4_K_M.gguf
```

可以拆成：

```text
Qwen-9B
└── Model

Q4_K_M
└── Quantization Type

GGUF
└── File Format
```

GGUF 文件中经常出现 Q4_K_M、Q5_K_M、Q6_K、Q8_0 等量化类型，但这些量化类型本身并不等于 GGUF。

## 3. safetensors

`safetensors` 是 Hugging Face 模型生态中非常常见的权重文件格式。

经常出现在：

- Hugging Face
- Transformers
- vLLM
- SGLang
- TensorRT-LLM
- ...

例如一个较大的模型可能包含：

```text
model-00001-of-00004.safetensors
model-00002-of-00004.safetensors
...
```

模型较大时，权重通常会被拆成多个文件。

## 4. GGUF 与 safetensors

可以粗略建立下面的常见生态关系：

```text
GGUF
 ↓
llama.cpp ecosystem
 ↓
CPU / Consumer Hardware / Local LLM


safetensors
 ↓
Hugging Face ecosystem
 ↓
Transformers / vLLM / SGLang / GPU Serving
```

| 文件格式 | 常见生态 | 常见场景 |
|---|---|---|
| GGUF | llama.cpp | CPU、消费级硬件、本地 LLM |
| safetensors | Hugging Face | Transformers、vLLM、SGLang、GPU Serving |

但这只是常见生态关系，并不是绝对限制。

## 5. 核心关系

理解模型文件时，最重要的是不要把不同层次混在一起：

```text
Model
  ↓
Model Weights
  ↓
Precision / Quantization
  ↓
Serialization / File Format
```

例如：

```text
Qwen3-8B-Q4_K_M.gguf
```

描述了：

- `Qwen3-8B`：模型
- `Q4_K_M`：量化类型
- `.gguf`：文件格式

而类似：

```text
Qwen-xxx-AWQ
```

其中 `AWQ` 描述的是量化方法，并不是文件格式。
