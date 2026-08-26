# 本地 LLM：模型基础

> 本文只讨论本地 LLM 体系中的“模型基础”部分，包括模型是什么、模型由什么组成、参数量代表什么，以及常见模型家族。

## 1. 什么是 LLM 模型

LLM（Large Language Model，大语言模型）本质上是一个经过大规模数据训练的神经网络模型。

在本地部署场景中，经常会看到不同的模型家族，例如：

- Qwen
- Llama
- DeepSeek
- GLM
- Gemma
- Mistral
- Phi
- ...

这些名称首先回答的是：

> **我们使用的是哪个模型或模型家族？**

例如 `Qwen3-8B` 可以简单拆成：

```text
Qwen3  → 模型家族 / 模型系列
8B     → 参数规模
```

模型名称本身和模型的精度、量化方式、文件格式以及推理框架是不同层次的概念。

## 2. 模型内容

一个可以实际加载和运行的 LLM，通常至少涉及：

```text
LLM Model
│
├── Model Architecture
├── Model Weights
├── Tokenizer
└── Configuration
```

| 组成 | 作用 |
|---|---|
| Model Architecture | 定义模型的网络结构 |
| Model Weights | 模型训练得到的大量参数 |
| Tokenizer | 负责文本与 Token 之间的转换 |
| Configuration | 保存模型结构和运行相关配置 |

### 2.1 Model Architecture

Model Architecture 描述模型的网络结构。对于现代 LLM，核心通常基于 Transformer 架构。

它决定了模型内部以下结构如何组织：

- Layer
- Attention
- Hidden Size
- Feed Forward Network
- Position Encoding
- ...

不同模型家族可能采用不同的 Transformer 设计和改进方案。

> **Architecture 可以简单理解为模型的结构设计。**

### 2.2 Model Weights

Model Weights 是模型训练得到的参数。

例如一个 `8B Model` 大约包含 **80 亿个参数**。这些参数保存了模型在训练过程中学习到的信息，模型文件中占用空间最大的部分通常就是 Model Weights。

模型部署时主要涉及：

- Disk Storage
- RAM
- VRAM

### 2.3 Tokenizer

LLM 并不是直接读取字符串，而是处理 Token。Tokenizer 负责文本与 Token ID 之间的转换：

```text
Text
 ↓
Tokenizer
 ↓
Token
 ↓
Token ID
 ↓
LLM
```

例如 `Hello world` 经过 Tokenizer 后，会被转换成模型能够处理的一组 Token ID。

模型生成结果时则执行相反过程：

```text
Token ID
 ↓
Tokenizer
 ↓
Text
```

因此 Tokenizer 是模型的重要组成部分，并且通常需要与对应模型匹配。

### 2.4 Configuration

Configuration 用来描述模型的一些结构和配置参数，例如：

- Model Type
- Hidden Size
- Number of Layers
- Attention Heads
- Vocabulary Size
- Context Length
- ...

这些信息帮助模型加载程序正确构建模型结构并加载对应权重。

## 3. 模型参数量

| 模型规模 | 大约参数量 |
|---:|---:|
| 4B | 40 亿 |
| 7B | 70 亿 |
| 8B | 80 亿 |
| 14B | 140 亿 |
| 32B | 320 亿 |
| 70B | 700 亿 |

### 3.1 参数量意味着什么

参数量是描述模型规模的重要指标。一般情况下，参数规模增加通常意味着：

```text
更多参数
   ↓
更大的模型权重
   ↓
更高的内存 / 显存需求
   ↓
更多的推理计算量
```

但是：

> **参数越多，并不意味着模型一定越好。**

模型实际能力还受到很多因素影响，例如：

- 模型架构
- 训练数据
- 训练方法
- Post-training
- 模型版本
- 任务类型
- ...

> 所以参数量更适合用于描述 **模型规模**，而不能单独作为模型能力的判断标准。

## 4. 参数量与模型权重大小

参数量决定了模型有多少参数，但模型实际占用多少存储空间，还取决于每个参数使用什么数据类型保存。

| 类型 | 每参数理论占用 | 8B 模型权重大小 |
|---|---:|---:|
| FP32 | 4 Bytes | ~32 GB |
| FP16 / BF16 | 2 Bytes | ~16 GB |
| FP8 | 1 Byte | ~8 GB |
| INT8 | 1 Byte | ~8 GB |
| INT4 | 0.5 Byte | ~4 GB |

因此，同样都是 `8B Model`，实际模型权重大小可能完全不同。例如：

```text
8B BF16  ≈ 16 GB 权重
8B 4-bit ≈  4 GB 权重
```

这里只是模型权重的粗略理论计算。

实际运行时还需要额外资源，例如：

- Model Weights
- KV Cache
- Runtime
- Temporary Memory
- Other Overhead

所以：

> **模型参数量不能直接等同于实际显存需求。**

Precision、Data Type 和 Quantization 会在独立的“模型精度与量化”文档中进一步讨论。

## 5. Dense Model 与 MoE Model

仅仅看到参数总量，有时仍然不足以判断模型实际推理成本。

现代 LLM 中常见两类结构：

- **Dense Model**
- **MoE Model**

### 5.1 Dense Model

Dense Model 可以简单理解为：

> 推理时模型的大部分参数都会参与计算。

例如一个 `8B Dense Model`，可以粗略理解为模型拥有约 **80 亿参数**，并且推理过程中这些参数构成主要计算主体。

### 5.2 MoE Model

MoE：

**Mixture of Experts**

MoE 模型通常包含多个 Expert，但每次处理 Token 时只激活其中一部分。

因此经常会看到：

```text
Total Parameters
        ≠
Activated Parameters
```

例如一个 MoE 模型可能拥有非常大的总参数量，但单次推理只激活其中部分参数。

所以比较 MoE 模型时，需要同时关注：

- Total Parameters
- Activated Parameters

而不能只看模型名称中的总参数规模。

## 6. Context Length

除了参数量之外，另一个经常出现在模型信息中的指标是：

**Context Length / Context Window**

它表示模型一次能够处理的上下文 Token 范围，例如：

- 32K
- 64K
- 128K
- ...

这里的 `K` 表示千级 Token 数量。

Context Length 与模型参数量是两个不同概念：

```text
Parameter Count → 模型有多少参数
Context Length  → 一次可以处理多少上下文 Token
```

更大的 Context Window 可以让模型一次处理更长的：

- Conversation
- Document
- Code
- Retrieved Context
- ...

但更长的上下文通常也会增加推理时的资源消耗，尤其会影响 KV Cache。

## 7. 常见模型家族

当前本地 LLM 生态中经常遇到的模型家族包括：

| 公司 / 组织 | 模型家族 | 典型方向 |
|---|---|---|
| Alibaba / Qwen Team | Qwen | 通用、中文、Coding、Reasoning、Agent |
| DeepSeek | DeepSeek | Reasoning、Coding、MoE |
| Z.ai / 智谱 | GLM | 通用、中文、Coding、Agent |
| Moonshot AI / 月之暗面 | Kimi | 长上下文、Coding、Agent |
| Google | Gemma | 通用、小中型模型、多模态生态 |
| Meta | Llama | 通用、开放权重生态 |
| Mistral AI | Mistral / Mixtral | 通用、效率、MoE |
| NVIDIA | Nemotron | Reasoning、Agent、企业部署 |
| Microsoft | Phi | 小模型、端侧部署 |

这里需要注意：

> 模型家族、具体模型版本和模型能力都会持续变化。

因此这张表主要用于认识模型生态，而不是作为固定的模型能力排名。
