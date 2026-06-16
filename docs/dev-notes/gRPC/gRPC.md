# gRPC 总结

## gRPC 基本介绍
gRPC（Google Remote Procedure Call）是 Google 开源的高性能 RPC 框架。它允许开发者像调用本地函数一样调用远程服务。

### RPC vs HTTP
#### 传统 HTTP 调用：
```text
客户端
  │
  │ GET /users/1001
  ▼
服务端
```
开发者需要关心：
- URL
- HTTP Method
- Header
- JSON序列化
- JSON反序列化

#### RPC 方式：
```
user_service.GetUser(1001)
```
开发者只关注：
- 调用哪个方法
- 传什么参数
- 返回什么结果

**网络通信细节由框架处理。**

## gRPC 的核心组成
### Proto 文件
用于定义：
- 服务(Service)
- 方法(RPC)
- 请求(Request)
- 响应(Response)

例如：
```proto
service UserService {
    rpc GetUser(GetUserRequest)
        returns (GetUserResponse);
}
```
### Protobuf
gRPC 默认使用 Protocol Buffers 作为序列化协议。

特点：
- 二进制格式
- 数据体积小
- 编解码速度快
- 强类型

### Code Generation
根据 proto 自动生成代码：
```text
user.proto
    │
    ▼
  protoc
    │
    ▼
  Python
   Java
    Go
   Node
    C#
```
无需手写：
- DTO
- 接口定义
- 序列化逻辑
  
## gRPC 的主要优势
- 高性能 -> HTTP/2 + Protobuf -> 网络开销更小，序列化更快
- 强类型 -> 编译期即可发现很多错误。
- 多语言支持 -> 同一个 proto可以生成：Python，Java，Go，Node，C++，C#
- 自动生成代码

## gRPC 在微服务中的作用
最常见用途：
```text
  服务A
    │
    ▼
  gRPC
    │
    ▼
  服务B
```
例如：
```text
Order Service
    │
    ├── User Service
    ├── Inventory Service
    └── Payment Service

服务之间通过 gRPC 通信。
```

## gRPC服务开发基本流程
```text
Proto Definition
        │
        ▼
      protoc
        │
        ▼
Generated Code
(Request / Response / Service Skeleton)
        │
        ▼
Service Implementation
(XXXServiceImpl)
        │
        ▼
Register Service
(server.py)
        │
        ▼
Add Interceptors
(Auth / Log / Trace)
        │
        ▼
Start gRPC Server
        │
        ▼
Receive RPC Requests
```