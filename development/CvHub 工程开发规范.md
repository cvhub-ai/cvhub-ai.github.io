# 工程开发规范

## 1. 适用范围

本规范适用于 CvHub 项目的日常研发、测试、代码评审、版本管理和发布过程。

本规范主要覆盖：

* 日志记录
* 异常处理
* 测试
* Git 提交
* 分支与合并
* Code Review
* 敏感信息管理
* 发布前检查

各组件和微服务在遵守本规范的同时，还应遵守：

* 微服务工具开发规范
* API 设计规范
* 跨语言统一开发规范
* 对应编程语言的具体开发规范

# 2. 日志规范

## 2.1 基本要求

所有服务和应用都必须使用统一的日志组件，不应通过 `print`、`Console.WriteLine` 等方式代替正式日志。

日志至少应包含：

* timestamp
* serviceName
* logLevel
* message

与请求处理相关的日志还应包含：

* requestId
* taskId（如有）
* processingTimeMs（如有）
* resultCount（如有）

推荐使用结构化日志。

示例：

```json
{
  "timestamp": "2026-07-31T12:30:15.245Z",
  "serviceName": "ocr-service",
  "logLevel": "INFO",
  "requestId": "req-123",
  "message": "OCR processing completed.",
  "processingTimeMs": 125.4,
  "resultCount": 3
}
```

## 2.2 日志级别

推荐使用以下日志级别：

### DEBUG

用于开发和调试阶段的详细信息，例如：

* 中间处理结果
* 配置解析结果
* 模块调用顺序
* 调试所需的尺寸、数量和状态

DEBUG 日志不应默认在生产环境中开启。

### INFO

用于记录正常运行过程中的关键事件，例如：

* 服务启动完成
* 模型加载完成
* 请求开始处理
* 请求处理完成
* 服务正常关闭

### WARNING

用于记录可以继续运行，但需要关注的问题，例如：

* 输入数据质量较低
* 使用默认配置代替缺失配置
* 外部依赖短暂不可用
* 请求接近资源限制
* 某个非关键处理步骤失败

### ERROR

用于记录当前操作失败，但服务仍可继续运行的问题，例如：

* 单次请求处理失败
* 模型推理失败
* 文件读取失败
* 外部服务调用失败

### CRITICAL

用于记录可能导致整个服务无法继续运行的问题，例如：

* 核心模型无法加载
* 必要配置缺失
* GPU 初始化失败
* 服务启动失败
* 核心资源不可恢复

## 2.3 请求日志

每个请求建议记录以下关键节点：

```text
Request received
→ Processing started
→ Processing completed / failed
```

请求开始时建议记录：

* requestId
* 接口名称
* 输入类型
* 输入数量
* 主要处理选项

请求结束时建议记录：

* requestId
* 处理结果
* processingTimeMs
* resultCount
* errorCode（如失败）

不建议为 Pipeline 中每个普通步骤都记录 INFO 日志。

内部步骤可以使用 DEBUG 日志，避免生产日志过于冗长。

## 2.4 禁止记录的内容

日志中禁止记录：

* 密码
* API Key
* Access Token
* Refresh Token
* 数据库密码
* 完整 Authorization Header
* 私钥
* 敏感配置文件内容
* 未经处理的用户隐私数据
* 大型二进制数据
* Base64 图片完整内容

如果确实需要记录输入信息，应只记录：

* 文件大小
* 图片尺寸
* 文件格式
* 哈希值
* 脱敏后的标识符

# 3. 异常处理规范

## 3.1 异常分类

异常至少应区分为：

* 输入错误
* 业务错误
* 模型错误
* 依赖错误
* 配置错误
* 系统错误

例如：

```text
InvalidInputError
UnsupportedImageFormatError
ModelLoadError
InferenceError
ExternalServiceError
ConfigurationError
```

## 3.2 异常信息

异常信息必须提供足够上下文。

推荐：

```text
Failed to load model 'ocr-recognizer' from '/models/ocr'.
```

不推荐：

```text
Failed
```

```text
Error
```

```text
Something went wrong
```

异常信息应尽量说明：

* 什么操作失败
* 哪个对象失败
* 失败发生在哪里
* 必要的标识信息

但不能包含敏感信息。

## 3.3 不得静默忽略异常

禁止：

```python
try:
    process()
except Exception:
    pass
```

异常必须至少进行以下一种处理：

* 转换为明确的业务异常
* 记录日志后重新抛出
* 返回统一错误结果
* 触发重试
* 标记任务失败

只有明确确认可以忽略的异常，才能忽略，并应增加注释说明原因。

## 3.4 异常转换

不同层之间应进行合理的异常转换。

例如：

```text
Paddle RuntimeError
        ↓
InferenceError
        ↓
gRPC INTERNAL
```

底层第三方异常不应直接暴露给外部调用方。

API 层负责将内部异常转换为：

* HTTP 状态码
* gRPC Status
* 统一错误结构

## 3.5 捕获范围

只捕获当前层能够处理的异常。

不推荐在底层模块中大量使用：

```python
except Exception:
```

应优先捕获具体异常类型。

全局异常处理器可以捕获未处理异常，但必须：

* 记录完整堆栈
* 返回统一错误响应
* 避免暴露内部实现细节

# 4. 测试规范

## 4.1 测试类型

每个服务至少应包含以下测试。

### Unit Tests

测试单个模块、类或函数。

例如：

* Validator
* Decoder
* Preprocessor
* Detector
* Recognizer
* Result Mapper
* Factory

Unit Test 应尽量：

* 执行速度快
* 不依赖真实网络
* 不依赖真实数据库
* 不加载大型模型
* 不依赖 GPU

### Integration Tests

测试多个模块之间的协作。

例如：

* Pipeline 与 Detector、Recognizer 的集成
* 模型加载与推理
* 数据库访问
* 文件系统
* 外部服务适配器

Integration Test 可以使用真实依赖，也可以使用专用测试环境。

### Service Tests

通过真实协议测试完整服务。

例如：

* 通过 gRPC 调用 Process
* 通过 HTTP 调用 Health
* 测试请求参数校验
* 测试错误响应
* 测试服务启动和关闭

Service Test 应验证外部调用方实际看到的行为。

## 4.2 测试命名

测试名称应体现：

* 测试对象
* 触发条件
* 预期结果

推荐：

```text
test_process_returns_empty_result_when_no_text_is_detected
```

```text
Process_ShouldReturnEmptyResult_WhenNoTextIsDetected
```

不推荐：

```text
test_process
```

```text
test_01
```

## 4.3 测试结构

推荐按照以下结构编写测试：

```text
Arrange
Act
Assert
```

示例：

```python
def test_run_returns_empty_result_when_detector_returns_no_regions():
    detector = FakeDetector(results=[])
    recognizer = FakeRecognizer()
    pipeline = OcrPipeline(detector, recognizer)

    result = pipeline.run(test_image)

    assert result == []
```

## 4.4 测试独立性

每个测试应：

* 可以独立运行
* 不依赖执行顺序
* 不共享可变全局状态
* 不依赖其他测试生成的数据
* 运行结束后清理临时资源

禁止通过调整测试顺序来解决测试失败。

## 4.5 Mock 和 Fake

对于以下依赖，可以使用 Mock 或 Fake：

* 外部服务
* 数据库
* 文件系统
* 模型推理
* GPU
* 消息队列

但不要对所有内部对象过度 Mock。

测试应关注行为和结果，而不是过度绑定内部实现细节。

## 4.6 测试数据

测试数据应：

* 尺寸尽量小
* 内容明确
* 可重复使用
* 不包含敏感数据
* 不依赖个人本地路径

大型模型和大型测试数据不应直接提交到 Git 仓库。

## 4.7 Bug 修复测试

修复 Bug 时，应尽量先增加一个能够复现问题的测试。

推荐流程：

```text
添加失败测试
→ 修复问题
→ 测试通过
→ 提交代码
```

# 5. Git Commit 规范

## 5.1 Commit 格式

统一使用：

```text
<type>: <description>
```

例如：

```text
feat: add OCR processing endpoint
fix: handle empty image input
refactor: simplify OCR engine construction
test: add pipeline integration tests
docs: update microservice development standard
```

## 5.2 Commit 类型

常用类型：

* `feat`：新增功能
* `fix`：修复问题
* `refactor`：代码重构，不改变外部功能
* `test`：新增或修改测试
* `docs`：文档修改
* `build`：构建系统或依赖修改
* `ci`：持续集成配置修改
* `chore`：其他维护工作
* `perf`：性能优化
* `style`：不影响逻辑的格式调整
* `revert`：撤销已有提交

## 5.3 Commit 描述

Commit 描述应：

* 简洁明确
* 使用英文
* 使用动词开头
* 描述本次提交的主要变化
* 避免句号结尾

推荐：

```text
fix: prevent repeated model loading
```

不推荐：

```text
fix: bug
```

```text
update code
```

```text
changes
```

## 5.4 Commit 粒度

一个 Commit 应只包含一个相对完整的逻辑修改。

推荐：

```text
refactor: extract detector factory
```

```text
test: add detector factory tests
```

不推荐将以下内容混在同一个 Commit：

* 大规模重构
* 新功能
* 文档更新
* 无关格式化
* Bug 修复

除非这些修改不可分割。

## 5.5 提交前检查

Commit 前至少检查：

* 代码可以正常运行
* 相关测试通过
* 没有调试代码
* 没有未使用代码
* 没有敏感信息
* 没有无关文件
* Commit 内容与描述一致

# 6. 分支与合并规范

## 6.1 主分支

推荐使用：

```text
main
```

`main` 分支应保持：

* 可以构建
* 可以测试
* 可以部署
* 不包含未完成代码

不应直接在 `main` 上进行大型功能开发。

## 6.2 功能分支

分支名称应体现修改目的。

推荐：

```text
feature/ocr-grpc-service
fix/model-loading-error
refactor/ocr-engine-factory
docs/development-standards
```

避免：

```text
test
new
my-branch
temp
```

## 6.3 分支生命周期

功能分支应：

* 从最新 `main` 创建
* 只处理一个明确任务
* 尽量保持较短生命周期
* 合并后及时删除

长期不合并的大型分支容易产生大量冲突，应尽量避免。

## 6.4 合并前要求

合并前必须：

* 更新目标分支最新代码
* 解决冲突
* 运行相关测试
* 完成 Code Review
* 检查 CI 状态
* 确认没有敏感信息

## 6.5 合并方式

GitHub 支持以下三种 Pull Request 合并方式：

### Squash Merge（推荐）

将 Pull Request 中的所有 Commit 合并为一个新的 Commit，再合并到目标分支。

**优点：**

- 保持 `main` 分支历史简洁。
- 一个 Pull Request 对应一个完整功能提交。
- 避免大量中间 Commit（如 `fix`、`refactor`、`test`）进入主分支。

### Merge Commit

保留 Pull Request 中的所有 Commit，并额外创建一个 Merge Commit。

**适用于：** 需要完整保留开发历史的项目。

### Rebase Merge

保留 Pull Request 中的所有 Commit，但不创建 Merge Commit，使提交历史保持线性。

**适用于：** 希望保持线性历史，同时保留完整开发过程的项目。

CvHub 推荐统一使用 **Squash Merge**，保持 `main` 分支历史清晰，并使每个 Pull Request 对应一个完整、独立的功能提交。

# 7. Pull Request 规范

Pull Request 应包含：

* 修改目的
* 主要修改内容
* 测试方式
* 可能影响的模块
* 已知限制
* 相关 Issue 或任务

示例：

```markdown
## Purpose

Refactor OCR engine creation to support configurable detectors and recognizers.

## Changes

- Remove BaseEngine subclasses
- Add detector and recognizer factories
- Inject components into OcrEngine
- Update factory tests

## Tests

- Unit tests
- Factory tests
- OCR pipeline tests
```

Pull Request 不应过大。

如果同时包含多个无关改动，应拆分为多个 Pull Request。

# 8. Code Review 规范

## 8.1 Review 目标

Code Review 主要关注：

* 正确性
* 可维护性
* 可读性
* 一致性
* 安全性
* 测试完整性

Code Review 不是只检查代码格式。

## 8.2 Review 检查项

### 代码设计

* 服务或模块职责是否清晰
* 是否存在不必要的复杂设计
* 是否过度使用继承
* 是否可以通过组合简化
* 模块之间是否存在循环依赖
* 是否破坏现有边界

### 代码质量

* 命名是否清晰
* 函数是否过长
* 是否存在重复代码
* 是否存在未使用代码
* 是否存在硬编码配置
* 注释是否解释原因而不是重复代码

### 异常和日志

* 异常是否被正确处理
* 是否存在静默忽略异常
* 错误信息是否包含上下文
* 是否记录必要日志
* 是否包含 requestId
* 是否泄露敏感信息

### 测试

* 是否增加必要测试
* Bug 修复是否有回归测试
* 测试是否覆盖主要分支
* 测试是否依赖执行顺序
* Mock 是否过度

### 性能和资源

* 是否重复加载模型
* 是否创建不必要的大对象
* 是否存在资源未释放
* 是否可能造成内存或显存泄漏
* 是否存在明显阻塞操作
* 是否需要限制并发

### 接口兼容性

* 是否修改公共接口
* 是否破坏现有调用方
* 是否需要版本升级
* 是否更新相关文档

## 8.3 Review 意见

Review 意见应：

* 明确指出问题
* 说明修改原因
* 尽量提供可执行建议
* 区分必须修改和建议修改

可以使用：

```text
Required:
```

```text
Suggestion:
```

```text
Question:
```

避免只写：

```text
不好
改一下
这里不对
```

# 9. 敏感信息管理

禁止将以下内容提交到 Git：

* `.env`
* API Key
* Access Token
* 密码
* 私钥
* 数据库连接密码
* 内部证书
* 生产环境配置
* 客户敏感数据
* 未脱敏日志

项目应提供：

```text
.env.example
```

`.env.example` 只包含：

* 配置名称
* 示例值
* 必要说明

不能包含真实密钥。

发现敏感信息已提交时，不能只删除当前文件，还应：

* 立即废弃并更换密钥
* 清理 Git 历史
* 检查日志和构建产物
* 通知相关人员