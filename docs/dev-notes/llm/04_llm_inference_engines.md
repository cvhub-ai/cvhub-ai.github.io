# 本地 LLM：推理框架与推理引擎

> 本文讨论模型文件准备完成后，如何通过 Inference Runtime / Engine 将模型真正运行起来，以及常见的 llama.cpp、Ollama、vLLM、SGLang、Transformers 等工具的定位。

## 1. 什么是 Inference Runtime / Engine

模型训练完成以后，本质上拥有：

- Model Architecture
- Model Weights
- Tokenizer / Config

还需要程序负责：

```text
Load Model
     ↓
Load Tokenizer
     ↓
Receive Prompt / Messages
     ↓
Transformer Forward
     ↓
Manage KV Cache
     ↓
Token Sampling
     ↓
Generate Output
```

负责这些工作的系统可以称为：

**Inference Runtime / Inference Engine**

常见工具包括：

- llama.cpp
- vLLM
- SGLang
- Transformers
- TensorRT-LLM
- ExLlamaV2
- MLX

Ollama 与这些工具存在功能重叠，但更接近封装好的本地模型运行与管理平台。

## 2. llama.cpp

`llama.cpp` 是本地 LLM 推理领域非常重要的 Runtime。

主要特点：

- GGUF 核心生态
- CPU 推理能力强
- 支持 GPU Offload
- 支持消费级 GPU
- 支持多种硬件平台
- 适合低资源本地运行

典型关系：

```text
GGUF
  ↓
llama.cpp
  ↓
CPU / GPU
  ↓
Text Output
```

llama.cpp 也可以通过 `llama-server` 提供 HTTP 服务：

```text
Application
     ↓
HTTP
     ↓
llama-server
     ↓
GGUF Model
```

## 3. Ollama

Ollama 与 llama.cpp、vLLM 的定位并不完全相同。

它更接近：

> **易用的本地模型运行、管理和服务平台。**

它提供：

- 模型下载
- 模型管理
- 模型加载 / 卸载
- 模型配置
- CLI
- Local API
- 本地服务管理

可以粗略理解为：

```text
Ollama
│
├── Model Management
├── API
├── Configuration
└── Inference Backend
        ↓
    Local Model
```

因此：

> **llama.cpp 更接近底层 Inference Runtime。**

> **Ollama 更接近封装好的 Local LLM Platform。**

## 4. vLLM

vLLM 的核心定位是：

> **High-performance GPU LLM Inference / Serving Engine**

它重点解决：

- GPU 利用率
- High Throughput
- Concurrent Requests
- Continuous Batching
- KV Cache Management
- Multi-GPU
- Model Serving

典型场景：

```text
User A ─┐
User B ─┤
User C ─┤
User D ─┘
         ↓
     API Server
         ↓
       vLLM
         ↓
 GPU / GPU Cluster
```

vLLM 的代表性技术之一是 **PagedAttention**，用于更加高效地管理 Attention 推理过程中涉及的 KV Cache 等资源。

工程部署中常见：

```text
Application
     ↓
OpenAI-compatible API
     ↓
vLLM
     ↓
Model
```

这样上层 Application 可以通过统一 API 调用模型，而不需要直接依赖模型内部实现。

## 5. SGLang

SGLang 同样属于高性能 LLM Serving / Runtime 生态。

主要关注：

- High-performance Serving
- Structured Generation
- Reasoning Workload
- Agent Workload
- GPU Inference Optimization
- Concurrent Requests

可以粗略理解：

```text
        GPU LLM Serving
              │
      ┌───────┴───────┐
      ↓               ↓
    vLLM           SGLang
```

两者存在明显的使用场景重叠。

> 具体性能、功能和模型支持会随着版本快速变化，因此不适合长期固定地认为其中一个一定优于另一个。

## 6. Hugging Face Transformers

Transformers 和 vLLM、Ollama 的定位并不完全一样。

它首先是完整的 Transformer 模型开发生态：

```text
Transformers
│
├── Model
├── Tokenizer
├── Generation
├── Training
├── Fine-tuning
└── Inference
```

非常适合：

- 模型研究
- 加载 Hugging Face 模型
- 修改模型结构
- Fine-tuning
- 推理实验
- 开发模型相关功能

如果目标是 Large-scale、High-concurrency、Production Serving，通常会进一步考虑专门的 Serving Engine。

## 7. TensorRT-LLM

TensorRT-LLM 是 NVIDIA 面向 LLM 推理优化的重要技术栈。

重点包括：

- NVIDIA GPU
- CUDA
- TensorRT
- Kernel Optimization
- Low Latency
- High Throughput
- Multi-GPU
- Data Center Deployment

整体方向：

```text
Model
  ↓
NVIDIA Inference Optimization
  ↓
TensorRT-LLM
  ↓
NVIDIA GPU
```

它通常更适合追求 NVIDIA 平台极致推理性能的部署环境。

## 8. ExLlamaV2

ExLlamaV2 主要面向 NVIDIA GPU 上的高效量化模型推理。

相关生态经常包括：

- EXL2
- GPTQ
- NVIDIA GPU
- Consumer GPU

它更偏向：

> **单机 NVIDIA GPU 上高效运行低 bit 模型。**

## 9. MLX

MLX 是 Apple 面向 Apple Silicon 的机器学习框架。

典型硬件包括：

- M1
- M2
- M3
- M4
- ...

Apple Silicon 使用 Unified Memory，因此其本地 LLM 部署方式和传统的 `CPU RAM + Discrete NVIDIA VRAM` 存在明显区别。

MLX 因此形成了自己的模型与量化生态。

## 10. 推理框架对比

| Runtime / Platform | 常见模型生态 | 主要定位 |
|---|---|---|
| llama.cpp | GGUF、K-quants | 本地、CPU、消费级硬件 |
| Ollama | 本地模型生态 | 简单运行、管理和提供本地 LLM |
| vLLM | HF、safetensors、BF16、AWQ、GPTQ、FP8 等 | GPU 高吞吐 Serving |
| SGLang | HF、BF16、FP16、FP8、AWQ 等 | 高性能 Serving / Agent Workload |
| Transformers | safetensors、BF16/FP16、bitsandbytes 等 | 模型开发、研究、实验 |
| TensorRT-LLM | FP16、BF16、FP8、INT8、INT4 等 | NVIDIA 高性能部署 |
| ExLlamaV2 | EXL2、GPTQ 等 | NVIDIA GPU 本地低 bit 推理 |
| MLX | MLX 模型及量化生态 | Apple Silicon |

具体支持情况会随版本变化，应以各项目当前文档为准。

## 11. llama.cpp、Ollama 与 vLLM

这三个概念非常容易混淆，可以粗略理解为：

```text
llama.cpp
→ 怎么高效地在本地硬件上运行模型

Ollama
→ 怎么方便地管理和使用本地模型

vLLM
→ 怎么高性能地把 GPU 模型作为服务提供出去
```

| 工具 | 核心定位 | 更强调 |
|---|---|---|
| llama.cpp | Local Inference Runtime | GGUF、CPU、GPU Offload、Consumer Hardware |
| Ollama | Local Model Platform | Easy Installation、Model Management、CLI、Local API |
| vLLM | High-performance GPU Serving Engine | GPU、Concurrency、Batching、KV Cache、Throughput |

三者存在功能重叠，但核心设计目标不同。
