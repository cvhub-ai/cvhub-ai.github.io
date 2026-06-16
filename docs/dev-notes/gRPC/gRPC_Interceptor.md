# gRPC Interceptor（拦截器）
Interceptor（拦截器）是 gRPC 提供的一种机制，用于在 RPC 调用的前后插入通用逻辑。

它的核心作用是：
- 把公共逻辑从业务代码中抽离出来
- 避免在每个 RPC 方法中重复编写相同代码。

## 示例
假设有一个用户服务：
```proto
service UserService {
    rpc GetUser(...)
        returns (...);

    rpc CreateUser(...)
        returns (...);

    rpc DeleteUser(...)
        returns (...);
}
```
如果不使用拦截器，python代码如下：
```python
def GetUser(...):
    verify_token()
    log_request()
    trace_request()
    ...

def CreateUser(...):
    verify_token()
    log_request()
    trace_request()
    ...

def DeleteUser(...):
    verify_token()
    log_request()
    trace_request()
    ...
``` 
会出现大量重复代码。

使用 Interceptor 后：
```text
请求
  │
  ▼
AuthInterceptor
  │
  ▼
LogInterceptor
  │
  ▼
TraceInterceptor
  │
  ▼
业务代码
```
业务代码只关注业务逻辑。

## Interceptor 的执行流程
服务端：
```text
Client
   │
   ▼
Interceptor #1
   │
   ▼
Interceptor #2
   │
   ▼
Interceptor #3
   │
   ▼
RPC Method
```

返回时：
```text
RPC Method
   │
   ▲
Interceptor #3
   │
   ▲
Interceptor #2
   │
   ▲
Interceptor #1
   │
   ▲
Client
```
类似责任链模式（Chain Of Responsibility）。

## 服务端拦截器（Server Interceptor）
常见用途：
- 认证
- 鉴权
- 日志
- 链路追踪
- 监控
- 异常处理
- 限流
- 熔断
  
## 客户端拦截器（Client Interceptor）
常见用途：
- 自动附加Token
- 自动附加TraceId
- 请求日志
- 超时控制
- 重试