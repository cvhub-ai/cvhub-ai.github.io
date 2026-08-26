# 本地 LLM：上层应用

> 本文讨论本地 LLM 部署体系中的 Application Layer，只说明 Chat、RAG、Agent 等上层应用与 LLM 的基本关系，不展开这些系统内部的具体实现。

## 1. Application Layer

LLM 部署完成后，上层才是真正使用模型能力的业务系统。

常见 Application 包括：

- Chat Application
- RAG
- Agent
- Coding Assistant
- Document Analysis
- Business Service

完整链路可以表示为：

```text
User
 ↓
Application
 ↓
LLM API
 ↓
Serving
 ↓
Inference Runtime
 ↓
Model
 ↓
CPU / GPU
```

Application 通常不需要直接操作模型文件或底层推理过程，而是通过 LLM API 使用模型能力。

## 2. Chat Application

Chat Application 是最直接的 LLM 应用形式。

基本关系：

```text
User
 ↓
Chat Application
 ↓
LLM API
 ↓
LLM
 ↓
Response
```

Application 负责用户交互，LLM 负责根据输入生成结果。

## 3. RAG

RAG 在调用 LLM 之前增加外部知识检索与 Context Building。

基本关系：

```text
User Query
    ↓
Retrieval
    ↓
Context Building
    ↓
LLM API
    ↓
LLM
    ↓
Answer
```

因此 RAG 仍然属于 LLM 的上层应用。

底层 LLM 可以通过统一 API 被 RAG 系统调用，而 RAG 本身负责检索和构建额外上下文。

## 4. Agent

Agent 同样建立在 LLM 之上，但除了调用模型之外，还可能涉及 Tool Calling 和外部服务。

基本关系：

```text
User
 ↓
Agent
 ↓
LLM API
 ↓
LLM
 ↓
Tool Calling / Reasoning
 ↓
External Tools / Services
```

因此 Agent 不是推理引擎，也不是模型文件格式，而是使用 LLM 能力构建的上层系统。

## 5. 其他业务应用

除了 Chat、RAG 和 Agent，LLM 还可以作为能力组件被其他应用调用，例如：

- Coding Assistant
- Document Analysis
- Business Service

它们在整体架构中的位置基本一致：

```text
Business Application
        ↓
      LLM API
        ↓
      Serving
        ↓
Inference Runtime
        ↓
       Model
```

具体业务逻辑由 Application 负责，底层 LLM 提供语言模型能力。

## 6. 整体关系

从本地模型到最终应用，可以形成完整链路：

```text
Model
  ↓
Precision / Quantization
  ↓
File Format
  ↓
Inference Runtime
  ↓
Serving
  ↓
Application
```

其中 Application Layer 主要解决：

> **部署好的 LLM 最终被谁使用，以及如何进入实际业务系统。**

Chat、RAG、Agent 等都属于这一层，而它们各自的内部架构可以作为独立主题继续讨论。
