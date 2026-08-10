# 跨语言统一开发规范

## 1. 适用范围

本规范适用于 CvHub 项目中使用不同编程语言实现的所有代码，包括但不限于：

* Python
* C#
* C++
* TypeScript

本规范用于统一跨语言的软件设计原则、命名方式和基础编码习惯。

各语言在遵守本规范的基础上，还应遵循对应语言自身的官方或团队规范。

## 2. 软件设计原则

### 2.1 模块职责单一

每个模块、类或组件应只负责一个明确职责。

推荐：

```text
Detector
Recognizer
Preprocessor
Postprocessor
Decoder
Validator
```

避免一个类同时负责：

- 读取配置
- 模型加载
- 数据预处理
- 模型推理
- 结果转换
- 日志输出

模块职责越清晰，越容易：

* 独立测试
* 独立替换
* 独立维护
* 复用

### 2.2 Pipeline 模块化

复杂处理流程应拆分为职责明确的模块，并由统一 Pipeline 负责组织。

例如：

```text
Input Validation
→ Decode
→ Preprocess
→ Inference
→ Postprocess
→ Result
```

每个模块应：

* 具有明确输入和输出
* 不依赖其他模块内部实现
* 可以独立测试
* 可以独立替换
* 不负责完整流程调度

Pipeline 负责：

* 模块调用顺序
* 数据传递
* 完整处理流程编排
* 对外提供统一入口

避免：

* 将完整流程写在一个大型函数中
* 模块之间形成复杂调用关系
* 在 API 层手动组织内部处理步骤

### 2.3 优先组合，谨慎继承

优先通过组合构建复杂能力。

推荐：

```text
Pipeline
├── Detector
├── Recognizer
├── Preprocessor
└── Postprocessor
```

例如：

```python
pipeline = OcrPipeline(
    detector=detector,
    recognizer=recognizer,
)
```

只有存在明确且稳定的 `is-a` 关系时才使用继承。

避免为了少量代码复用构建复杂继承层级。

### 2.4 依赖抽象而不是具体实现

当一个模块存在多个可替换实现时，应优先依赖稳定接口或抽象类型。

例如：

```text
TextDetector
    ↑
    ├── PaddleTextDetector
    └── CraftTextDetector
```

Pipeline 应依赖：

```text
TextDetector
```

而不是直接依赖：

```text
PaddleTextDetector
```

具体实现由 Factory 或依赖注入机制负责构建。

### 2.5 显式优于隐式

重要逻辑、配置和处理步骤应明确表达。

推荐：

```text
Input
→ Validate
→ Decode
→ Process
→ Result
```

避免：

* 隐藏关键处理流程
* 依赖大量全局状态
* 使用难以理解的默认行为
* 在函数内部偷偷创建复杂依赖

代码应尽可能让调用关系和数据流清晰可见。

## 3. 命名规范

### 3.1 类和类型

类、接口、枚举、异常统一使用：

```text
UpperCamelCase
```

例如：

```text
OcrPipeline
ImageProcessor
ProcessingRequest
ProcessingResult
ModelLoadError
ProcessingStatus
```

### 3.2 函数、方法和变量

跨语言逻辑命名统一使用：

```text
lowerCamelCase
```

例如：

```text
loadModel
processImage
requestId
modelPath
processingResult
confidenceThreshold
```

具体语言如果存在强制或主流命名规范，可以在语言适配规范中调整。

例如 Python 会使用 `snake_case`。

### 3.3 常量

常量统一使用：

```text
UPPER_SNAKE_CASE
```

例如：

```text
DEFAULT_TIMEOUT_SECONDS
MAX_IMAGE_SIZE_BYTES
SUPPORTED_IMAGE_FORMATS
```

### 3.4 布尔变量

布尔变量或判断函数应使用具有判断语义的前缀：

```text
is...
has...
can...
should...
```

例如：

```text
isModelLoaded
hasValidInput
canProcessRequest
shouldApplyRotation
```

避免：

```text
flag
check
statusFlag
value
```

### 3.5 集合命名

集合使用复数形式：

```text
images
textRegions
processingResults
modelNames
```

不要使用：

```text
imageList
resultList
dataArray
```

除非数据结构类型本身对业务语义非常重要。

映射关系应体现 Key 和 Value：

```text
modelByName
resultByRequestId
serviceByCapability
```

### 3.6 缩写命名

缩写按照普通单词处理。

推荐：

```text
OcrPipeline
HttpClient
JsonSerializer
GrpcServer
ApiRequest
GpuDevice
```

避免项目中混用：

```text
OCRPipeline
OcrPipeline
ocrPipeline
```

同一缩写必须保持统一。

### 3.7 名称表达业务含义

变量名称应表达其真实业务含义。

推荐：

```text
confidenceThreshold
processingResult
detectedRegions
modelConfig
```

不推荐：

```text
data
temp
obj
value
info
result2
```

短生命周期局部变量除外。

## 4. 参数设计

### 4.1 避免含义不清晰的布尔参数

不推荐：

```python
processImage(image, True, False, True)
```

调用方无法直接理解每个参数的含义。

推荐使用明确参数：

```python
processImage(
    image=image,
    enableRotation=True,
    enableResize=False,
)
```

如果参数存在多个明确状态，优先使用枚举。

例如：

```python
options = ImageProcessingOptions(
    rotationMode=RotationMode.AUTO,
    resizeMode=ResizeMode.DISABLED,
)
```

### 4.2 参数数量应保持合理

函数参数过多通常意味着：

* 函数职责过多
* 配置没有合理封装
* 数据结构设计不清晰

当多个参数属于同一业务概念时，应封装为 Options、Config 或 Request 对象。

例如：

```text
ProcessingOptions
ModelConfig
ImageProcessingOptions
```

### 4.3 不使用魔法数字

不推荐：

```python
if confidence < 0.5:
```

推荐：

```python
if confidence < MIN_CONFIDENCE_THRESHOLD:
```

或者：

```python
if confidence < config.confidenceThreshold:
```

重要阈值应：

* 使用常量
* 使用配置
* 或明确解释来源

## 5. 函数和方法设计

### 5.1 单一职责

一个函数应完成一个明确任务。

如果函数同时包含：

```text
读取文件
→ 解析配置
→ 加载模型
→ 推理
→ 保存结果
```

通常应该拆分。

### 5.2 控制函数长度

不设置绝对行数限制，但函数应保持容易理解。

当函数出现以下情况时应考虑拆分：

* 多层嵌套
* 多个不同处理阶段
* 大量局部变量
* 多个不同错误处理逻辑
* 难以用一句话描述函数职责

### 5.3 减少嵌套

优先使用提前返回。

不推荐：

```python
if request is not None:
    if request.image is not None:
        if is_valid(request.image):
            process(request.image)
```

推荐：

```python
if request is None:
    return

if request.image is None:
    return

if not isValid(request.image):
    return

process(request.image)
```

### 5.4 避免隐藏副作用

函数名称应能够反映其行为。

例如：

```text
loadModel()
saveResult()
deleteFile()
```

不应该在：

```text
getResult()
```

中偷偷：

* 写数据库
* 删除文件
* 修改配置
* 初始化模型

## 6. 数据模型

### 6.1 使用明确的数据结构

跨模块传递数据时，应优先使用明确的数据模型。

例如：

```text
ProcessingRequest
ProcessingResult
DetectionResult
RecognitionResult
BoundingBox
```

避免大量使用：

```text
dict
Dictionary<string, object>
Map<string, any>
tuple
```

来传递核心业务数据。

### 6.2 输入输出保持稳定

模块之间的输入和输出结构应尽可能稳定。

修改公共模型时，应考虑：

* 是否影响其他模块
* 是否影响 API
* 是否需要兼容旧字段
* 是否需要更新测试

### 6.3 DTO 与内部模型分离

外部通信使用的数据结构不应直接等同于内部业务模型。

例如：

```text
gRPC Request
        ↓
DTO / Mapper
        ↓
Internal Model
```

这样可以避免通信协议变化直接影响内部逻辑。

## 7. 配置管理

### 7.1 配置与代码分离

以下内容不应硬编码在业务代码中：

* 模型路径
* 服务地址
* GPU 设备
* 日志级别
* 超时时间
* 最大批量数量
* 文件大小限制

应通过：

```text
Environment Variables
Config File
Validated Config Object
```

统一管理。

### 7.2 配置集中读取

环境变量和配置文件应集中读取。

不推荐：

```text
module A → getenv()
module B → getenv()
module C → getenv()
```

推荐：

```text
Environment
    ↓
Config Loader
    ↓
Validated Config
    ↓
Modules
```

### 7.3 提供默认值时应明确

默认值必须：

* 合理
* 可解释
* 有明确单位
* 不隐藏关键行为

例如：

```text
DEFAULT_TIMEOUT_SECONDS
MAX_BATCH_SIZE
MAX_IMAGE_SIZE_BYTES
```

## 8. 错误与异常设计

### 8.1 使用明确异常类型

推荐：

```text
InvalidInputError
ModelLoadError
InferenceError
ConfigurationError
```

避免所有错误都使用：

```text
Exception
RuntimeError
```

### 8.2 错误信息必须包含上下文

推荐：

```text
Failed to load model 'ocr-recognizer' from '/models/ocr'.
```

不推荐：

```text
Load failed.
```

### 8.3 不静默忽略异常

禁止：

```python
try:
    process()
except Exception:
    pass
```

异常必须：

* 被处理
* 被转换
* 被记录
* 或重新抛出

## 9. 注释和文档

### 9.1 注释解释“为什么”

不推荐：

```python
# Increase index
index += 1
```

推荐：

```python
# Skip the background class because it is not part of the output labels.
index += 1
```

代码本身应该尽可能说明“做什么”，注释主要解释：

* 为什么这样做
* 特殊约束
* 算法假设
* 非显而易见的设计选择

### 9.2 公共接口需要文档

公共类、公共方法和重要模块应说明：

* 功能
* 输入
* 输出
* 异常
* 必要限制

尤其是：

```text
Pipeline
Detector
Recognizer
Factory
Service Interface
```

## 10. 代码一致性

同一个项目中，相同问题应采用一致解决方式。

例如：

* 相同类型的配置使用相同结构
* 相同类型的异常使用相同模式
* 相同类型的 Factory 使用相似设计
* 相同缩写采用相同命名
* 相同模块遵循相似目录结构

不要因为个人偏好在不同模块中采用完全不同的设计风格。

## 11. 避免过度设计

只有存在明确需求时才增加：

* 抽象层
* Factory
* Adapter
* Repository
* Interface
* Event System
* Plugin System

不要为了“以后可能会用”提前构建复杂架构。

但如果已经明确存在：

* 多个可替换实现
* 多种后端
* 明确扩展点
* 第三方依赖隔离需求

则应提前建立稳定抽象。

原则是：

> 保持简单，但保留明确边界。

## 12. 开发检查项

提交代码前至少检查：

* 命名是否清晰并符合规范
* 模块职责是否单一
* Pipeline 是否承担完整流程编排
* 是否存在不必要的继承
* 是否可以通过组合简化
* 是否依赖具体实现而不是抽象
* 是否存在魔法数字
* 是否存在含义不清晰的布尔参数
* 函数是否过长或嵌套过深
* 数据模型是否明确
* 配置是否被硬编码
* 异常是否被静默忽略
* 是否存在不必要的复杂设计
* 相似模块是否保持统一风格
