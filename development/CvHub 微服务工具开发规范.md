# 微服务工具开发规范

## 1. 适用范围

本规范适用于 CvHub 项目中的所有独立微服务工具，包括但不限于：

- OCR Service
- Object Detection Service
- Feature Matching Service
- QR / Barcode Service
- ...
  
本规范用于统一各微服务在`职责划分`、`接口设计`、`目录结构`、`Pipeline 组织`、`资源管理`和`服务生命周期`等方面的实现方式。

各服务在遵守本规范的基础上，还应遵守：

- 跨语言统一开发规范
- API 设计规范
- 工程开发规范
- 对应编程语言的具体开发规范

## 2. 核心原则

### 2.1 单一职责

一个微服务只提供一个相对完整、独立的业务能力。

例如：

- OCR Service：完成完整 OCR 流程
- Object Detection Service：完成目标检测流程
- Feature Matching Service：完成特征匹配流程
- QR / Barcode Service：完成二维码和条形码识别流程

不要将多个彼此无关的能力放入同一个服务。

服务边界应按照`完整业务能力`划分，而不是按照单个模型、类或算法步骤划分。

例如，OCR 内部可以包含：

- 图像预处理
- 文本检测
- 文本识别
- 结果后处理

这些步骤共同组成完整的 OCR 能力，不应被拆分为多个独立微服务。

### 2.2 服务接口保持精简

每个服务原则上只提供少量稳定、明确的外部接口。

基础接口建议包括：

- Health
- SingleProcess
- BatchProcess
- GetCapabilities
- GetServiceInfo

对于异步任务，可以根据需要增加：

- GetTaskStatus
- CancelTask

**不要将 Pipeline 内部的每个处理阶段暴露为独立接口**

例如，OCR Service 不应对外分别暴露：

- PreprocessImage
- DetectText
- RecognizeText
- PostprocessResult

外部调用方只需要描述需要完成的任务，不需要了解服务内部的算法实现和处理步骤。

### 2.3 业务逻辑与通信协议分离

HTTP、gRPC 或消息队列只负责通信，不负责核心业务逻辑。

推荐分层：
```text
API / gRPC Layer
        ↓
Application Layer
        ↓
Domain / Pipeline Layer
        ↓
Infrastructure Layer
```

各层职责应清晰分离。核心业务逻辑不应直接依赖通信协议层。

协议层负责将外部请求转换为服务内部使用的数据模型，再调用 Application 或 Pipeline。

错误示例：
```python
def process(request: GrpcProcessingRequest):
    image = decode_image(request.image_data)
    model = load_model(request.model_path)
    result = model.predict(image)
    return GrpcProcessingResponse(...)
```

推荐示例：
```python
def process(request: GrpcProcessingRequest):
    processing_request = request_mapper.to_domain(request)
    processing_result = processing_service.process(processing_request)
    return response_mapper.to_grpc(processing_result)
```

这样可以保证：

- 核心逻辑可以脱离通信协议独立测试
- 可以同时支持 gRPC、HTTP 或本地调用
- 更换框架时不需要重写业务逻辑
- 服务内部模型不会被协议对象污染

### 2.4 Pipeline 模块化与明确边界

完整处理流程应拆分为**职责独立**、**边界清晰**的模块，并由统一 Pipeline 负责组织。

典型流程：
```text
输入验证
→ 数据解码
→ 数据预处理
→ 模型推理
→ 结果后处理
→ 结果输出
```

每个模块应满足：

- 只负责一个明确的处理阶段
- 具有清晰的输入和输出
- 不依赖其他模块的内部实现
- 可以独立测试
- 可以独立替换
- 不负责完整流程调度

Pipeline 负责：

- 按照确定顺序调用各个模块
- 在模块之间传递数据
- 组织完整业务处理流程
- 对外提供统一、稳定的处理入口
- 处理流程级异常和上下文信息

推荐调用方式：
```python
result = processing_pipeline.run(request)
```

外部调用方不需要了解 Pipeline 内部的处理细节。

应避免：

- 将所有处理逻辑堆积在一个大型函数中
- 模块之间直接调用并形成复杂依赖
- 在 API 或 gRPC 层中手动组织 Pipeline 步骤
- 将内部处理阶段全部暴露为外部接口
- 在每个请求中动态创建全部处理模块

### 2.5 优先组合，谨慎继承

服务内部模块优先通过组合进行组装。

例如：
```text
ProcessingPipeline
├── Decoder
├── Validator
├── Preprocessor
├── InferenceEngine
└── Postprocessor
```
只有在存在明确、稳定的“is-a”关系时才使用继承。

不应为了复用少量代码而构建复杂继承层级。

推荐：
```python
pipeline = OcrPipeline(
    detector=detector,
    recognizer=recognizer,
    preprocessor=preprocessor,
)
```

组合方式通常具有以下优势：

- 模块替换更简单
- 测试更方便
- 依赖关系更明确
- 避免继承层级过深
- 更适合通过 Factory 或 Dependency Injection 进行构建

## 3. 服务目录职责

不同语言的具体目录名称可以不同，但逻辑职责应保持一致。

推荐逻辑结构：
```text
service/
├── api/
├── application/
├── domain/
├── infrastructure/
├── pipeline/
├── config/
└── tests/
```

目录名称可以根据语言生态调整，但不能失去职责边界。

### 3.1 API

**API 层负责服务与外部调用方之间的通信** 

主要职责：

- 接收请求
- 参数解析
- 输入格式检查
- 协议对象转换
- 调用 Application 层
- 返回响应
- 处理 HTTP 状态码或 gRPC Status

API 层不负责：

- 模型推理
- 图像处理
- Pipeline 编排
- 数据库业务逻辑
- 加载模型
- 复杂业务判断

API 层应尽可能薄。

### 3.2 Application

**Application 层负责组织具体用例**

例如：

- 处理单张图片
- 批量处理
- 查询任务状态
- 取消任务
- 调用 Pipeline
- 组织多个领域能力完成一个用例

Application 层负责“做什么”，但不应包含底层算法实现。

示例：
```
ProcessImageUseCase
BatchProcessingUseCase
GetTaskStatusUseCase
CancelTaskUseCase
```

### 3.3 Domain
**Domain 层负责服务内部稳定的核心模型，也就是核心业务的数据结构部分**

包括：

- 核心数据模型
- 输入输出模型
- 业务规则
- 枚举
- 异常
- 抽象接口
- 能力定义

例如：
```text
OcrRequest
TextDetector (ABC)
TextRecognizer (ABC)
DetectionResult
RecognitionResult
OcrResult
```

Domain 层不应依赖：

- Web 框架
- gRPC 框架
- 数据库实现
- 文件系统实现
- 第三方服务客户端

Domain 层应尽可能保持纯净。

### 3.4 Infrastructure

**Infrastructure 层负责所有具体技术实现和外部依赖**

包括：

- 模型加载
- 推理框架适配
- GPU 和硬件访问
- 数据库
- 文件系统
- 对象存储
- 第三方库
- 外部服务
- 消息队列
- 缓存

例如：
```text
PaddleTextDetector
YoloInferenceEngine
LocalFileStorage
PostgreSqlTaskRepository
GrpcModelClient
```
Domain 层定义抽象接口，Infrastructure 层提供具体实现。

### 3.5 Pipeline

**Pipeline 层负责组织完整处理流程**

主要职责：

- 调用输入验证
- 调用数据解码
- 调用预处理
- 调用模型推理
- 调用后处理
- 生成统一结果
- 记录流程级上下文
- 处理流程级错误

Pipeline 不应负责：

- 解析 HTTP 或 gRPC Request
- 直接创建数据库连接
- 每次请求重新加载模型
- 读取未封装的全局配置
- 返回框架特定 Response

### 3.6 Config

**Config 负责服务运行配置**

配置内容可以包括：

- 模型路径
- 服务端口
- 日志级别
- GPU 设备
- Worker 数量
- 超时时间
- 队列大小
- 功能开关
- 配置应在服务启动时读取和校验。
- 业务代码不应在多个位置直接读取环境变量。

推荐：
```text
Environment Variables
        ↓
Configuration Loader
        ↓
Validated Service Config
        ↓
Service Components
```

### 3.7 Tests

**Tests 目录负责服务测试**

建议包括：
```text
tests/
├── unit/
├── integration/
└── service/
```
具体测试要求由工程开发规范统一定义。

## 4. 服务接口标准

### 4.1 Health

每个服务必须提供健康检查接口。

健康检查建议区分：

#### Liveness
用于判断服务进程是否正常运行。

检查内容应尽量简单，例如：

- 服务进程是否可响应
- 主事件循环是否正常
- 服务是否处于不可恢复状态

Liveness 不应执行耗时模型推理，也不应因为临时外部依赖故障频繁失败。

#### Readiness

用于判断服务是否已经准备好接收请求。

可以检查：

- 模型是否加载完成
- 模型预热是否完成
- GPU 是否可用
- 必要依赖是否可用
- 必要配置是否有效
- 服务是否正在关闭

推荐状态：
```text
STARTING
READY
NOT_READY
STOPPING
FAILED
```
Health 接口应快速返回，不应触发实际业务处理。

### 4.2 GetCapabilities

每个服务应能够描述自身能力。

示例：
```json
{
  "serviceName": "ocr-service",
  "version": "1.0.0",
  "capabilities": [
    "textDetection",
    "textRecognition"
  ],
  "supportedFormats": [
    "jpg",
    "png",
    "pdf"
  ]
}
```

GetCapabilities 可以包含：

- serviceName
- version
- capabilities
- supportedFormats
- supportedLanguages
- supportedModels
- processingModes
- limits
- optionalFeatures

Capabilities 可用于：

- Agent 选择工具
- 服务注册
- 运行时能力检查
- 兼容性判断
- 自动生成文档
- 调试和运维检查

Capabilities 应描述稳定能力，不应暴露内部敏感配置。

不应返回：

- 绝对模型路径
- API Key
- Access Token
- 内部网络地址
- 数据库连接信息
- 服务器敏感硬件信息

### 4.3 Process

Process 是服务的主要处理接口。

请求只应包含完成任务所需的业务参数，例如：

- requestId
- 输入数据
- confidenceThreshold
- language
- processingOptions
- outputOptions

不允许调用方修改服务内部运行配置，例如：

- modelPath
- gpuDevice
- workerCount
- logLevel
- internalCacheSize
- modelLoadStrategy

内部配置应由服务部署环境和配置文件控制，而不是由单次请求决定。

Process 接口应满足：

- 输入含义清晰
- 输出结构稳定
- 可追踪
- 可验证
- 不暴露内部模块
- 不允许随意控制内部执行流程

### 4.4 异步任务接口

当单次处理耗时较长，或需要队列调度时，可以提供异步接口。

常见接口：

- SubmitTask
- GetTaskStatus
- GetTaskResult
- CancelTask

任务状态可以包括：
```text
PENDING
RUNNING
SUCCEEDED
FAILED
CANCELLED
```
异步接口应提供稳定的 taskId 或 requestId。

是否使用异步模式应根据任务特点决定，不应为了“架构完整”而强行增加任务系统。

## 5. 模型和资源管理

AI 模型、GPU 上下文、线程池、数据库连接池等都属于重量级资源。

这些资源应：

- 在服务启动阶段初始化
- 在整个服务生命周期内复用
- 在服务关闭阶段释放
- 初始化失败时阻止服务进入 Ready 状态

禁止：

- 每个请求重复加载模型
- 每个请求重新创建 GPU 上下文
- 在业务函数中随意创建连接池
- 依赖对象长期隐藏在全局变量中且无法管理
- 服务未准备完成时接收业务请求

推荐生命周期：
```text
读取配置
→ 初始化日志
→ 校验运行环境
→ 初始化基础设施
→ 加载模型
→ 模型预热
→ 启动通信服务
→ Readiness = READY
→ 接收请求
→ Readiness = STOPPING
→ 停止接收新请求
→ 等待正在处理的请求完成
→ 释放资源
→ 服务关闭
```

### 5.1 模型加载

模型加载应集中管理。

推荐由专门组件负责：

- ModelManager
- ModelLoader
- InferenceEngine

模型加载失败时，应输出明确上下文，例如：
```text
Failed to load model 'ocr-recognizer' from '/models/ocr'.
```
模型加载状态应能够被 Readiness 检查使用。

### 5.2 模型预热

对于首次推理延迟较高的模型，服务启动时应进行预热。

预热可以用于：

- 初始化 GPU Kernel
- 初始化推理图
- 分配显存
- 验证模型可正常推理
- 避免首个真实请求延迟过高
- 预热失败时，服务不应进入 Ready 状态。

### 5.3 并发与线程安全

服务必须明确模型推理对象是否支持并发调用。

需要考虑：

- 推理引擎是否线程安全
- 是否需要锁
- 是否需要请求队列
- 是否使用多个 Worker
- GPU 显存是否足够
- 是否允许并行批处理
- 是否需要限制最大并发数

不要默认认为第三方模型对象一定线程安全。

### 5.4 资源限制

服务应对资源使用设置明确限制。

例如：

- 最大输入文件大小
- 最大图片尺寸
- 最大批量数量
- 最大并发请求数
- 最大等待队列长度
- 最大处理时间
- 最大显存占用策略

超出限制时应返回明确错误，而不是让服务失控或崩溃。

### 5.5 优雅关闭

服务关闭时应：

- 停止接收新请求
- 将 Readiness 设置为不可用
- 等待正在处理的请求完成或超时
- 取消可安全取消的任务
- 关闭连接池
- 释放模型和 GPU 资源
- 刷新日志
- 保存必要状态

不能直接终止进程而忽略正在执行的请求。

## 6. 服务构建与依赖注入

服务内部组件应在启动阶段统一构建。

推荐：
```text
Configuration
    ↓
Component Factory / Dependency Injection
    ↓
Infrastructure Components
    ↓
Pipeline
    ↓
Application Service
    ↓
API / gRPC Server
```

例如：
```python
detector = detector_factory.create(config.detector)
recognizer = recognizer_factory.create(config.recognizer)

pipeline = OcrPipeline(
    detector=detector,
    recognizer=recognizer,
)

processing_service = ProcessingService(pipeline=pipeline)
```

应避免：

- 在 API Handler 中创建模型
- 在 Pipeline 内部读取环境变量
- 在业务方法中硬编码具体实现
- 通过大量全局变量共享依赖
- 让模块自行查找和创建所有依赖

统一构建有助于：

- 清晰管理依赖
- 替换具体实现
- 编写测试
- 管理资源生命周期
- 避免重复初始化

## 7. 服务边界判断

新增功能时，应先判断它是否应该：

- 作为现有服务内部模块
- 作为现有服务的新能力
- 作为独立微服务
- 作为通用基础设施组件

适合成为独立微服务的能力通常满足：
- 具有完整、独立的业务目标
- 可以通过稳定接口调用
- 可以独立部署
- 可以独立扩缩容
- 具有独立资源需求
- 生命周期与其他能力不同
- 失败不会必然导致其他服务不可用

不适合单独拆分的情况：

- 只是 Pipeline 中一个内部步骤
- 只被单个服务使用
- 输入输出高度依赖另一个模块内部结构
- 拆分后会产生大量频繁网络调用
- 没有独立部署和扩缩容需求
- 无法形成稳定接口

## 8. 禁止事项

微服务开发中禁止：

- 一个服务包含多个无关业务能力
- 将每个内部处理步骤暴露为外部接口
- 在 API 或 gRPC 层实现核心业务逻辑
- 在每个请求中重新加载模型
- 允许请求方修改服务内部运行配置
- 核心 Domain 直接依赖通信框架
- Pipeline 直接返回 HTTP 或 gRPC Response
- 模块之间形成复杂循环依赖
- 服务未完成初始化就进入 Ready 状态
- 健康检查执行耗时业务推理
- 在代码中硬编码 API Key、模型路径或环境配置
- 未限制输入大小和并发数量
- 忽略服务关闭时的资源释放

## 9. 开发检查清单

开发新的微服务时，至少确认：

- 服务边界
  - 服务是否只负责一个完整、独立的能力
  - 是否避免将内部算法步骤拆成多个微服务
  - 是否具有清晰稳定的输入和输出
- 接口
  - 是否提供 Health
  - 是否提供 GetCapabilities
  - 是否提供统一 Process 接口
  - 是否避免暴露 Pipeline 内部步骤
  - 是否禁止请求方修改内部配置
- 分层
  - API 层是否只负责通信和转换
  - Application 层是否负责组织用例
  - Domain 层是否独立于具体框架
  - Infrastructure 层是否封装外部依赖
  - Pipeline 是否负责统一流程编排
- 资源
  - 模型是否在服务启动时加载
  - 模型是否在请求之间复用
  - 是否执行模型预热
  - 是否明确并发和线程安全策略
  - 是否设置资源限制
  - 是否支持优雅关闭
- 可维护性
  - 组件是否通过组合构建
  - 依赖是否集中创建和管理
  - 配置是否集中读取和校验
  - 各模块是否可以独立测试和替换