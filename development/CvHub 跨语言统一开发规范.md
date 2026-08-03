# 1. 适用范围

本规范适用于 CvHub 项目中的所有组件，包括：

- AI Agent
- Microservice
  - OCR Service
  - Object Detection Service
  - Feature Matching Service
  - QR / Barcode Service
  - ...
- Python、C#、TypeScript、C++ 等不同语言实现

各语言必须先遵守本规范，再遵守对应语言的适配规范。


# 2. 微服务开发规范 - 核心原则

## 2.1 单一职责

一个服务只提供一个相对完整、独立的视觉能力。

例如：

- OCR Service：完成完整 OCR 流程
- Object Detection Service：完成目标检测流程
- Feature Matching Service：完成特征匹配流程

不要把多个无关能力放进同一个服务。

## 2.2 接口保持精简

每个视觉服务原则上只提供少量稳定接口：

```
Health
GetCapabilities
Process
```

必要时可以增加：

```
GetTaskStatus
CancelTask
```

**不要把 Pipeline 中的每个内部步骤都暴露成独立 API。**

例如 OCR 内部可以包含：

- 图像预处理
- 文本检测
- 文本识别
- 结果后处理

但对外只暴露完整的 OCR 处理接口。

## 2.3 业务逻辑与通信协议分离

HTTP、gRPC、消息队列只负责通信，不负责核心业务逻辑。

推荐结构：

```
API / gRPC Layer
        ↓
Application Layer
        ↓
Domain / Pipeline Layer
        ↓
Infrastructure Layer
```

核心 Pipeline 不应直接依赖：

- FastAPI
- ASP.NET Core
- gRPC Request
- HTTP Request
- 数据库连接对象

## 2.4 Pipeline 模块化与明确边界

完整的处理流程应拆分为职责独立、边界清晰的模块，并由统一的 Pipeline 负责模块编排。

例如：

```
输入验证
→ 数据解码
→ 数据预处理
→ 模型推理
→ 结果后处理
→ 结果输出
```

每个模块应：

- 只负责一个明确的处理阶段；
- 具有清晰的输入和输出；
- 不依赖其他模块的内部实现；
- 可以独立测试和替换；
- 不直接负责完整流程的调度。

Pipeline 负责：

- 按照确定顺序调用各个模块；
- 在模块之间传递数据；
- 组织完整的业务处理流程；
- 对外提供统一、稳定的处理入口。

外部调用方不需要了解 Pipeline 内部的所有处理细节，只需调用完整处理接口：

```
result = processingPipeline.run(request)
```

避免：

- **将所有处理逻辑堆积在一个大型函数中**
- **模块之间直接调用并形成复杂依赖**
- **在 API 或 gRPC 层中手动组织 Pipeline 的内部步骤**
- **将内部处理阶段全部暴露为外部接口**

## 2.5 优先组合，谨慎继承

优先通过组合组装不同模块：

```
Pipeline
├── Decoder
├── Preprocessor
├── Inference Engine
└── Postprocessor
```

只有存在明确且稳定的继承关系时才使用继承。

# 3. 跨语言命名规范

## 3.1 类和类型

类、接口、枚举、异常统一使用：

```
UpperCamelCase
```

例如：

```
OcrPipeline
ImageProcessor
ProcessingRequest
ProcessingResult
ModelLoadError
ProcessingStatus
```

## 3.2 函数、方法和变量

统一使用：

```
lowerCamelCase
```

例如：

```
loadModel
processImage
requestId
modelPath
processingResult
confidenceThreshold
```

## 3.3 常量

统一使用：

```
UPPER_SNAKE_CASE
```

例如：

```
DEFAULT_TIMEOUT_SECONDS
MAX_IMAGE_SIZE_BYTES
SUPPORTED_IMAGE_FORMATS
```

## 3.4 布尔命名

布尔变量或判断函数使用：

```
is...
has...
can...
should...
```

例如：

```
isModelLoaded
hasValidInput
canProcessRequest
shouldApplyRotation
```

禁止使用含义不清晰的名称：

```
flag
check
statusFlag
```

## 3.5 集合命名

集合使用复数：

```
images
textRegions
processingResults
modelNames
```

映射关系应体现键值：

```
modelByName
resultByRequestId
serviceByCapability
```

## 3.6 缩写命名

缩写按照普通单词处理：

```
OcrPipeline
HttpClient
JsonSerializer
GrpcServer
ApiRequest
```

项目中必须保持一致，不要混用：

```
OCRPipeline
OcrPipeline
ocrPipeline
```

# 4. API 和数据格式

## 4.1 JSON 字段

所有服务的 JSON 字段统一使用：

```
lowerCamelCase
```

例如：

```json
{
  "requestId": "req-123",
  "processingTimeMs": 125.4,
  "resultCount": 3
}
```

## 4.2 gRPC 字段

gRPC Message 字段统一使用：

```
lowerCamelCase
```

例如：

```protobuf
message ProcessingRequest {
  string requestId = 1;
  float confidenceThreshold = 2;
}
```

## 4.3 时间和单位

名称中必须写明单位。

推荐：

```
timeoutSeconds
processingTimeMs
fileSizeBytes
memoryLimitMb
```

不推荐：

```
timeout
duration
size
limit
```

## 4.4 统一错误结构

所有服务使用统一错误格式：

```json
{
  "errorCode": "INVALID_IMAGE_FORMAT",
  "message": "The image format is not supported.",
  "requestId": "req-123",
  "details": {}
}
```

字段含义：

```
errorCode   稳定错误代码
message     人类可读错误信息
requestId   请求追踪标识
details     可选附加信息
```

# 5. 服务目录职责

不同语言的具体目录名称可以不同，但逻辑职责应保持一致。

- API
- Application
- Domain
- Infrastructure
- Pipeline
- Config
- Tests

## 5.1 API

负责：

- 接收请求
- 参数解析
- 协议转换
- 状态码或 gRPC 状态处理

不负责核心业务逻辑。

## 5.2 Application

负责组织具体用例：

- 处理图片
- 批量处理
- 查询任务
- 调用 Pipeline

## 5.3 Domain

负责：

- 核心数据模型
- 业务规则
- 枚举
- 异常
- 抽象接口

## 5.4 Infrastructure

负责：

- 模型加载
- 数据库
- 文件系统
- 第三方库
- 外部服务

## 5.5 Pipeline

- 负责组织完整处理流程。


# 6. 服务接口标准

## 6.1 Health

每个服务必须提供健康检查。

建议区分：

Liveness：
- 服务进程是否正常运行

Readiness：
- 模型是否加载完成
- GPU 是否可用
- 必要依赖是否正常

## 6.2 GetCapabilities

服务应能够描述自身能力，例如：

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

Capabilities 用于：

- Agent 选择工具
- 服务注册
- 运行时检查
- 文档生成

## 6.3 Process

处理接口只接收完成任务所需参数。

请求参数可以包括：

- requestId
- 输入数据
- confidenceThreshold
- language
- processingOptions
- ...

不允许请求方修改服务内部配置，例如：

- modelPath
- gpuDevice
- workerCount
- logLevel

# 7. 模型和资源管理

AI 模型属于重量级资源，必须：

- 服务启动时加载
- 整个服务生命周期内复用
- 服务关闭时释放

禁止每个请求重复加载模型。

推荐生命周期：

```
读取配置
→ 初始化日志
→ 检查运行环境
→ 加载模型
→ 模型预热
→ 启动服务
→ 接收请求
→ 优雅关闭
```

# 8. 日志规范

所有服务日志至少包含：

- timestamp
- serviceName
- logLevel
- requestId
- message

处理请求时建议记录：

- requestId
- 处理开始时间
- 处理结束时间
- 处理耗时
- 结果数量
- 错误信息

禁止记录：

- 密码
- API Key
- Access Token
- 完整敏感数据

# 9. 异常处理

异常应分为：

- 输入错误
- 业务错误
- 模型错误
- 依赖错误
- 系统错误

错误信息必须提供上下文。

推荐：

```
Failed to load model 'ocr-recognizer' from '/models/ocr'.
```

不推荐：

```
Failed
Error
Something went wrong
```

异常不能被静默忽略。

# 10. 测试规范

每个服务至少包含：

- Unit Tests
  - 测试单个模块或类
- Integration Tests
  - 测试 Pipeline 和模型
  - 测试数据库或外部依赖
- Service Tests
  - 通过 HTTP 或 gRPC 测试完整服务

测试名称应体现：

- 测试对象
- 触发条件
- 预期结果

# 11. Git 规范

## 11.1 Commit 格式

```
<type>: <description>
```

例如：

```
feat: add OCR processing endpoint
fix: handle empty image input
refactor: split preprocessing pipeline
test: add model loading tests
docs: update gRPC interface
```

常用类型：

- feat
- fix
- refactor
- test
- docs
- build
- ci
- chore
- perf

# 12. Code Review 检查项

提交前检查：

- 命名是否符合统一规范
- 服务职责是否单一
- API 是否过度拆分
- 业务逻辑是否进入 API 层
- 模型是否重复加载
- 异常是否被静默忽略
- 日志是否包含 requestId
- 是否存在敏感信息
- 是否有必要的测试
- 不同服务的数据结构是否一致