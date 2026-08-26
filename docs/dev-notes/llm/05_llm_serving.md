# 本地 LLM：LLM Serving

> 本文讨论模型已经能够通过推理引擎运行以后，如何将模型能力以服务的形式提供给其他程序。

## 1. 什么是 Serving

推理引擎负责：

> **运行模型。**

Serving Layer 负责：

> **让其他程序调用模型。**

整体关系可以理解为：

```text
Application
     ↓
LLM API
     ↓
Serving
     ↓
Inference Engine
     ↓
Model
```

## 2. 常见 API 形式

Serving Layer 常见的接口形式包括：

- HTTP API
- REST API
- OpenAI-compatible API
- gRPC

它们的核心作用都是在 Application 和底层模型推理之间建立稳定的调用接口。

## 3. OpenAI-compatible API

工程部署中经常会看到：

```text
Application
     ↓
OpenAI-compatible API
     ↓
LLM Serving
     ↓
Inference Engine
     ↓
Model
```

OpenAI-compatible API 的价值在于统一上层调用方式。

例如以下应用：

- Agent
- RAG
- Chat Backend
- Business Service

可以依赖统一 API，而不需要直接关心底层模型到底由哪个 Runtime 执行。

## 4. Serving 与高并发推理

在模型 Serving 场景中，可能同时存在多个用户请求：

```text
User A ─┐
User B ─┤
User C ─┤
User D ─┘
         ↓
     API Server
         ↓
Inference / Serving Engine
         ↓
       GPU
```

因此高性能 Serving Engine 会重点关注：

- GPU 利用率
- Concurrent Requests
- Continuous Batching
- KV Cache Management
- Throughput
- Multi-GPU

vLLM 和 SGLang 都属于这一类高性能 GPU LLM Serving / Runtime 生态。

## 5. Serving 与 Application 的关系

Serving Layer 位于推理引擎与业务应用之间：

```text
Model
  ↓
Inference Runtime
  ↓
Serving / API
  ↓
Application
```

这样可以把模型运行方式与业务系统解耦。

上层 Application 只需要知道：

```text
调用哪个 API
发送什么 Request
接收什么 Response
```

而不需要直接处理：

- 模型文件加载
- GPU 推理
- KV Cache
- Token Sampling
- 模型底层 Runtime

因此 Serving 是从“模型能够运行”到“模型能够被业务系统使用”之间的重要一层。
