# gRPC Code Generation & Service Implementation

## 核心思想
```text
.proto
   ↓
生成服务接口骨架
   ↓
实现这个接口
   ↓
注册到 gRPC Server
```
对于服务的具体实现，不同的语言需要选择不同的语言特性

## Python：继承基类
### Proto文件
```proto
service UserService {
    rpc GetUser(GetUserRequest)
        returns (GetUserResponse);
}
```

### 生成类
```python
class UserServiceServicer:

    def GetUser(self, request, context):
        raise NotImplementedError()
```

### Python实现
```python
class UserServiceImpl(
        UserServiceServicer):

    def GetUser(self, request, context):
        ...
```
> **属于：`继承 + 重写方法`**

## Java：继承抽象基类
### Proto生成类
```Java
public abstract class UserServiceImplBase {

    public void getUser(
        GetUserRequest request,
        StreamObserver<GetUserResponse> responseObserver
    ) {
        ...
    }
}
```

### Java实现
```Java
public class UserServiceImpl
    extends UserServiceGrpc.UserServiceImplBase {

    @Override
    public void getUser(
        GetUserRequest request,
        StreamObserver<GetUserResponse> responseObserver
    ) {

        ...
    }
}
```
> **属于：`继承 + 重写方法`**

## Go：实现接口（最典型）
### Proto生成
```Go
type UserServiceServer interface {

    GetUser(
        context.Context,
        *GetUserRequest,
    ) (*GetUserResponse, error)
}
```

### go实现
```Go
type UserServiceImpl struct {
}

func (s *UserServiceImpl) GetUser(
    ctx context.Context,
    req *GetUserRequest,
) (*GetUserResponse, error) {

    ...
}
```

### 注册：
```Go
pb.RegisterUserServiceServer(
    grpcServer,
    &UserServiceImpl{},
)
```

> **这里没有继承。属于：`接口实现`**

## C#
### Proto生成
```C#
public abstract class UserServiceBase
{
    public virtual Task<GetUserResponse>
        GetUser(
            GetUserRequest request,
            ServerCallContext context)
}
```

### C#实现
```C#
public class UserServiceImpl
    : UserService.UserServiceBase
{
    public override Task<GetUserResponse>
        GetUser(...)
    {
        ...
    }
}
```
> **属于：`继承 + 重写方法`**

## Node.js
Node 更灵活。

### Proto生成 Service Definition
```JavaScript
{
  GetUser: {
     path: "/user.UserService/GetUser"
  }
}
```

### Node实现
```JavaScript
function getUser(call, callback) {

    callback(null, {
        name: "Tom"
    });
}
```

### 注册
```JavaScript
server.addService(
    UserService.service,
    {
        GetUser: getUser
    }
)
```
> **属于：`函数注册` 没有继承**

## Rust（Tonic）
### Proto生成 Trait：
```Rust
#[tonic::async_trait]
pub trait UserService {

    async fn get_user(
        &self,
        request: Request<GetUserRequest>,
    ) -> Result<Response<GetUserResponse>>;
}
```

### Rust实现
```Rust
impl UserService for UserServiceImpl {

    async fn get_user(...) {

    }
}
```
> **属于：`Trait` 实现**

## 总结
可以把 gRPC 的服务实现理解成：
```text
gRPC定义接口
        ↓
各语言生成服务骨架
        ↓
开发者实现骨架
        ↓
注册到Server
```

不同语言只是"骨架"的表现形式不同：
<table style="border-collapse: collapse;">
  <thead>
    <tr>
      <th style="border: 1px solid; padding: 8px;">语言</th>
      <th style="border: 1px solid; padding: 8px;">生成内容</th>
      <th style="border: 1px solid; padding: 8px;">实现方式</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="border: 1px solid; padding: 8px;">Python</td>
      <td style="border: 1px solid; padding: 8px;">Servicer 基类</td>
      <td style="border: 1px solid; padding: 8px;">继承</td>
    </tr>
    <tr>
      <td style="border: 1px solid; padding: 8px;">Java</td>
      <td style="border: 1px solid; padding: 8px;">ImplBase 抽象类</td>
      <td style="border: 1px solid; padding: 8px;">继承</td>
    </tr>
    <tr>
      <td style="border: 1px solid; padding: 8px;">C#</td>
      <td style="border: 1px solid; padding: 8px;">Base 抽象类</td>
      <td style="border: 1px solid; padding: 8px;">继承</td>
    </tr>
    <tr>
      <td style="border: 1px solid; padding: 8px;">Go</td>
      <td style="border: 1px solid; padding: 8px;">Interface</td>
      <td style="border: 1px solid; padding: 8px;">实现接口</td>
    </tr>
    <tr>
      <td style="border: 1px solid; padding: 8px;">Rust</td>
      <td style="border: 1px solid; padding: 8px;">Trait</td>
      <td style="border: 1px solid; padding: 8px;">实现 Trait</td>
    </tr>
    <tr>
      <td style="border: 1px solid; padding: 8px;">Node</td>
      <td style="border: 1px solid; padding: 8px;">Service Definition</td>
      <td style="border: 1px solid; padding: 8px;">注册函数</td>
    </tr>
  </tbody>
</table>