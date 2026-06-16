# gRPC Proto
## Proto 文件（接口契约定义）
Proto 文件本质上是：IDL(Interface Definition Language) 接口定义语言

Proto 文件只负责：
- 定义接口

不会包含：
- 业务逻辑
- 数据库访问
- 缓存操作
这些内容由具体语言实现。

## Protobuf3 数据类型详解
### 基本语法
```proto
syntax = "proto3";
```

每个字段都有编号：
```proto
message User {
    int64 id = 1;
    string name = 2;
}
```
字段编号就是参数占位，一旦上线尽量不要修改。

### 基本类型
#### 整数
```proto
int32 age = 1;
int64 user_id = 2;
uint32 count = 3;
uint64 total = 4;
```

#### 浮点数
```proto
float score = 1;
double price = 2;
```

#### 布尔
```proto
bool enabled = 1;
```

#### 字符串
```proto
string name = 1;
```

二进制
```proto
bytes content = 1;
```
适合：
- 图片
- 文件
- 加密数据

### Message（对象类型）
Message 相当于一个类。
```proto
message User {
    int64 id = 1;
    string name = 2;
}
```
#### 嵌套 Message
```proto
message Address {
    string city = 1;
    string street = 2;
}

message User {
    int64 id = 1;
    Address address = 2;
}
```

也可以直接嵌套定义：
```proto
message User {
    message Address {
        string city = 1;
    }
    Address address = 1;
}
```

### repeated（集合）
表示数组或列表。
```proto
message User {
    repeated string tags = 1;
}
```
对应：list[str]

对象集合：
```proto
message User {
    repeated Address addresses = 1;
}
```
对应：list[Address]

### Enum（枚举）
```proto
enum UserStatus {
    UNKNOWN = 0;
    ACTIVE = 1;
    DISABLED = 2;
}
```

使用：
```proto
message User {
    UserStatus status = 1;
}
```
注意：
- 建议始终保留第一个枚举值为 0 -> UNKNOWN = 0

### Map（映射）
```proto
map<string,string> labels = 1;
map<int64, User> users = 2;
```

Key支持类型：
```proto
bool
string

int32
int64

uint32
uint64

sint32
sint64

fixed32
fixed64

sfixed32
sfixed64
```

Key不支持：
```proto
map<float, User>     // ❌

map<double, User>    // ❌

map<bytes, User>     // ❌

map<MyEnum, User>    // ❌

map<MyMessage, User> // ❌
```

### Oneof（互斥字段）
同一时刻只能有一个字段生效。
```proto
message SearchRequest {
    oneof condition {
        string name = 1;
        int64 user_id = 2;
    }
}
```
合法：`name` 或者 `user_id`, 但这两个参数不会同时存在

### Import（文件引用）
公共结构通常会抽到单独 proto。
```proto
// common.proto
message PageRequest {
    int32 page = 1;
    int32 size = 2;
}

//user.proto
import "common.proto";
message UserQuery {
    PageRequest page = 1;
}
```

### Package
用于命名空间管理。
```proto
package user;
```
避免大型项目中的名称冲突。

### Service
定义服务接口。
```proto
service UserService {
    rpc GetUser(
        GetUserRequest
    ) returns (
        GetUserResponse
    );

    rpc CreateUser(
        CreateUserRequest
    ) returns (
        CreateUserResponse
    );
}
```
Service 会生成对应语言的：
- Client Stub
- Server Base Class