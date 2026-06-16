# gRPC Communication Patterns
## Unary RPC（单请求单响应）
- 客户端发送一次请求：Request
- 服务端返回一次响应：Response

### 流程
```text
Client
   │ Request
   ▼
Server
   │ Response
   ▼
Client
```

### Proto定义
```proto
service UserService {
    rpc GetUser(
        GetUserRequest
    ) returns (
        GetUserResponse
    );
}
```

### 适用场景
- 查询用户
- 创建订单
- 删除商品
- 登录认证

绝大多数微服务调用都是这种模式。

## Server Streaming RPC（服务端流式）
- 客户端发一次请求。
- 服务端返回多个响应。

### 流程
```text
Client
   │ Request
   ▼
Server
   │ Response #1
   │ Response #2
   │ Response #3
   ▼
Client
```

### Proto定义
```proto
service LogService {
    rpc TailLog(
        LogRequest
    ) returns (
        stream LogMessage
    );
}
```
- 注意：`stream LogMessage` 表示返回流。

### 适用场景
- 日志实时查看
- 监控数据推送
- 大数据导出
- 实时事件通知

## Client Streaming RPC（客户端流式）
- 客户端发送多个请求。
- 服务端最后返回一个响应。

### 流程
```text
Client
   │ Request #1
   │ Request #2
   │ Request #3
   ▼
Server
   │ Response
   ▼
Client
```

### Proto定义
```proto
service FileService {

    rpc UploadFile(
        stream FileChunk
    ) returns (
        UploadResponse
    );
}
```
注意：`stream FileChunk`

### 适用场景
- 文件上传
- 批量数据导入
- 客户端日志上报
- IoT设备数据上传

## Bidirectional Streaming RPC（双向流）
- 客户端和服务端都可以持续发送数据,互不等待。

### 流程
```text
    Client
     ▲ |      
MsgA | │ Msg1
MsgB | │ Msg2
MsgC | │ Msg3     
     │ ▼  
    Server
```
双方同时发送。

### Proto定义
```proto
service ChatService {
    rpc Chat(
        stream ChatMessage
    ) returns (
        stream ChatMessage
    );
}
```

### 适用场景
- 聊天室
- 实时协同编辑
- 在线游戏
- 股票行情
- 实时AI对话
- 语音通话
- 如何记忆

## 微服务项目中的使用频率
- Unary RPC                90%+
- Server Streaming         少量
- Client Streaming         极少
- Bidirectional Streaming  极少