# 本地 LLM：模型精度与量化

> 本文讨论本地 LLM 中的模型精度与量化，包括 Precision / Data Type、量化的目的，以及 K-quants、AWQ、GPTQ、FP8 等常见概念。

## 1. Precision / Data Type

模型权重需要使用某种数值类型保存。常见类型包括：

- FP32
- FP16
- BF16
- FP8
- INT8
- INT4

粗略来看：

| 类型 | 每参数理论占用 | 8B 模型权重大小 |
|---|---:|---:|
| FP32 | 4 Bytes | ~32 GB |
| FP16 / BF16 | 2 Bytes | ~16 GB |
| FP8 | 1 Byte | ~8 GB |
| INT8 | 1 Byte | ~8 GB |
| INT4 | 0.5 Byte | ~4 GB |

以上只是模型权重的粗略理论占用。实际运行时还需要额外资源，例如：

- KV Cache
- CUDA / Runtime
- 临时计算空间
- 其他内存开销

> **8B BF16 模型的权重大约 16 GB，并不意味着 16 GB 显存一定能够完整运行。**

### 1.1 FP16 与 BF16

FP16 和 BF16 都是 16-bit 浮点类型，常用于：

- GPU 推理
- 模型训练
- Fine-tuning
- 高质量模型 Serving

对于现代 LLM，BF16 是非常常见的高精度部署形式之一。

### 1.2 FP8

FP8 是 **8-bit Floating Point**。

FP8 首先描述的是一种低精度浮点表示 / 计算体系，而 AWQ、GPTQ 更接近具体的量化方法，因此它们并不完全处于同一个技术层次。

典型关系可以简单理解为：

```text
Modern GPU
    ↓
FP8 Weight / Computation
    ↓
Tensor Core / GPU Optimization
    ↓
Higher Throughput + Lower Resource Usage
```

FP8 更常见于支持相应低精度计算能力的现代 GPU 和数据中心部署环境，经常与以下技术栈一起出现：

- vLLM
- SGLang
- TensorRT-LLM

## 2. 什么是 Quantization

量化的核心目标是：

> **使用更低精度表示模型参数，从而减少模型大小、显存 / 内存占用，并在适合的硬件和推理框架上提高推理效率。**

一般趋势：

```text
压缩程度提高
      ↓
资源占用降低
      ↓
潜在模型能力损失风险提高
```

实际效果并不是简单线性关系，还取决于：

- 模型本身
- 量化算法
- bit 数
- Quantization Group
- Calibration
- 推理框架
- 硬件

## 3. Quantization Method 与 Scheme

量化相关概念可以进一步区分为 **Quantization Method** 和 **Quantization Scheme / Type**。

### 3.1 Quantization Method

Quantization Method 描述模型**如何被量化**，例如：

- AWQ
- GPTQ
- SmoothQuant
- bitsandbytes
- ...

### 3.2 Quantization Scheme / Type

Quantization Scheme / Type 描述量化权重具体采用什么表示方式。

例如 llama.cpp / GGUF 生态中的：

- Q4_K_M
- Q5_K_M
- Q6_K
- Q8_0

因此下面这些概念不能简单理解成同一种分类：

```text
FP8     → Precision / Data Type
AWQ     → Quantization Method
GPTQ    → Quantization Method
Q4_K_M  → Quantization Scheme / Type
GGUF    → File Format
```

## 4. K-quants

K-quants 是 llama.cpp / GGUF 生态中常见的一类量化体系。

常见类型包括：

- Q4_K_S
- Q4_K_M
- Q5_K_S
- Q5_K_M
- Q6_K
- Q8_0

其中可以粗略理解：

```text
Q4 → 主要为约 4-bit
Q5 → 主要为约 5-bit
Q6 → 主要为约 6-bit
Q8 → 主要为约 8-bit
```

`S` 和 `M` 常见于：

```text
S = Small
M = Medium
```

可以粗略理解为不同的混合量化策略。不同 Tensor 可以使用不同精度，在 Model Size 与 Model Quality 之间取得平衡。

| Quantization | 特点 |
|---|---|
| Q4_K_M | 体积和质量比较均衡，常见选择 |
| Q5_K_M | 占用更高，通常质量更好 |
| Q6_K | 更大，进一步接近高精度模型 |
| Q8_0 | 占用明显更高，量化损失通常较小 |

实际 GGUF 量化模型还存在：

- Block Quantization
- Scale
- Metadata
- 不同 Tensor 使用不同精度

因此模型实际大小并不是简单的 `参数量 × bit 数`。

## 5. AWQ

AWQ：

**Activation-aware Weight Quantization**

它属于权重量化方法。核心思想之一是利用 activation 信息判断哪些权重更加重要，从而尽量降低低 bit 权重量化造成的能力损失。

典型流程：

```text
Original Transformer Model
        ↓
       AWQ
        ↓
Quantized Weights
        ↓
GPU Inference Engine
        ↓
vLLM / SGLang / ...
```

AWQ 常见于：

- NVIDIA GPU
- GPU Server
- 高性能推理
- LLM Serving

模型仓库中经常会看到类似 `Qwen-xxx-AWQ` 的模型名称。

## 6. GPTQ

GPTQ 是经典的 **Post-Training Quantization**，即在模型训练完成以后进行量化。

基本流程：

```text
Trained Model
      ↓
     GPTQ
      ↓
Low-bit Quantized Weights
      ↓
GPU Inference
```

GPTQ 长期以来被广泛用于 Transformer 模型的低 bit 推理，相关生态包括：

- GPTQModel
- ExLlama / ExLlamaV2
- Transformers
- 部分 vLLM 场景

## 7. 其他量化技术

除了 K-quants、AWQ、GPTQ、FP8，还可能遇到：

- bitsandbytes
- torchao
- EXL2
- INT8
- SmoothQuant
- ...

这些技术并不全部属于相互竞争的同类方案，它们往往针对不同的：

- Hardware
- Runtime
- Model
- Deployment Scenario

可以先用下面的关系进行粗略理解：

```text
消费级电脑 / CPU / 混合推理
        ↓
GGUF + Q4/Q5/Q6
        ↓
llama.cpp

消费级 / Server NVIDIA GPU
        ↓
AWQ / GPTQ / BF16
        ↓
vLLM / SGLang / ExLlamaV2

现代数据中心 GPU
        ↓
BF16 / FP8
        ↓
vLLM / SGLang / TensorRT-LLM
```

实际支持情况应以具体模型、硬件和 Runtime 的当前版本为准。
