

# TTDb Protobuf 三个文件深度解析

> 基于项目实际代码分析，涵盖 PubSubMessage.proto、SubConvertible.proto、SubProtocol.proto 三个文件的设计思路、字段含义、文件作用和调用链路。

---

## 一、整体定位与设计思路

项目的 IM 通信采用 **三层 Protobuf 协议架构**，三个 .proto 文件各司其职：

```
┌─────────────────────────────────────────────────────────┐
│  PubSubMessage.proto  —  服务端 Pub/Sub 架构协议         │
│  （iOS 客户端当前未直接使用，服务端内部通信协议）        │
├─────────────────────────────────────────────────────────┤
│  SubProtocol.proto   —  WebSocket 通信核心协议          │
│  （客户端主干协议，所有 IM 消息都走这套协议）            │
├─────────────────────────────────────────────────────────┤
│  SubConvertible.proto —  业务数据转换协议               │
│  （历史消息拉取等具体业务的响应格式，嵌套在 SubProtocol 内） │
└─────────────────────────────────────────────────────────┘
```

**为什么要分三个文件？**

1. **职责分离** — 传输层协议（SubProtocol）和业务数据模型（SubConvertible）分开，各自独立演进
2. **跨端复用** — PubSubMessage 是服务端 Pub/Sub 架构的标准协议，iOS / Android / 后端共用定义
3. **可扩展性** — 新增业务数据格式只需新增 .proto 文件，不改动核心传输协议
4. **包名区分** — `app.meeky`（服务端） vs `app.ios`（iOS 侧业务模型），清晰标识归属

---

## 二、PubSubMessage.proto 详解

### 2.1 文件基本信息

| 项 | 值 |
|---|---|
| 文件路径 | `TTDb/Protobuf/PubSubMessage.proto` |
| 生成文件 | `TTDb/Classes/Protobuf/PubSubMessage.pb.swift` |
| 语法 | proto3 |
| package | `app.meeky` |
| Swift 前缀 | `IM` |
| iOS 端使用情况 | ❌ 业务代码零引用（服务端协议，客户端未接入） |

### 2.2 为什么定义这个文件

这是服务端 **发布-订阅（Pub/Sub）架构** 的协议定义。服务端的消息分发系统是基于 Pub/Sub 模型构建的：
- 不同的通道（Channel）负责不同类型的消息投递
- 监听器（Listener）订阅通道，接收推送
- 内部消息用于连接管理和监听器管理

iOS 客户端当前走的是另一条路（SubProtocol 的 WebSocket request/response 模式），但保留了这套协议定义，可能原因：
1. 服务端内部使用这套协议分发消息，iOS 端通过 WebSocket 接收的消息最终也是从 Pub/Sub 系统出来的
2. 未来可能接入原生 Pub/Sub 客户端 SDK
3. 多端协议对齐，Android 或其他端可能直接使用

### 2.3 各 message 详解

#### 2.3.1 PubSubMessage — 顶层消息信封

```proto
message PubSubMessage {
    Type type = 1;
    bytes content = 2;

    enum Type {
        UNKNOWN = 0;
        DELIVER = 1;          // 消息投递
        KEEPALIVE = 2;        // 心跳保活
        CLOSED = 3;           // 连接关闭
        CONNECT_CHANGED = 4;  // 连接状态变化
        INTERNAL = 5;         // 服务器内部消息
    }
}
```

**设计思路**：典型的 **type + content** 信封模式。外层只有两个字段：
- `type` — 告诉接收方 content 里装的是什么，按类型反序列化
- `content` — 二进制消息体，序列化后的子消息

**为什么 content 用 bytes 而不是 oneof？**
- proto3 早期 oneof 支持不够完善时的经典设计
- 向后兼容性好：新增类型不需要修改信封结构
- 服务端转发时不需要解析 content，只看 type 即可

| 字段 | 类型 | 编号 | 作用 |
|---|---|---|---|
| type | Type enum | 1 | 消息类型，决定 content 的解析方式 |
| content | bytes | 2 | 二进制消息体，根据 type 反序列化为子消息 |

#### 2.3.2 InternalMessage — 服务器内部消息

```proto
message InternalMessage {
    Type type = 1;
    bytes content = 2;

    enum Type {
        UNKNOWN = 0;
        CONNECTED = 1;   // 连接已建立
        LISTENER = 2;    // 监听器操作
    }
}
```

**设计思路**：服务端内部组件之间通信用的消息。当 PubSubMessage.type = INTERNAL 时，content 反序列化为 InternalMessage。

#### 2.3.3 ConnectedMessage — 连接建立成功

```proto
message ConnectedMessage {
    string userId = 1;
    string connectId = 2;
    uint32 loginType = 3;
}
```

**触发时机**：客户端 WebSocket 连接建立成功后，服务端 Pub/Sub 系统内部发出的通知。

| 字段 | 类型 | 作用 |
|---|---|---|
| userId | string | 连接对应的用户 ID |
| connectId | string | 本次连接的唯一标识（多端同时在线时区分不同连接） |
| loginType | uint32 | 登录类型 / 设备类型（如 iOS=1, Android=2, Web=3） |

#### 2.3.4 ConnectChangedMessage — 连接状态变化

```proto
message ConnectChangedMessage {
    string userId = 1;
    bool connected = 2;
}
```

**触发时机**：用户上线或下线时，Pub/Sub 系统发出的状态变更通知。

| 字段 | 类型 | 作用 |
|---|---|---|
| userId | string | 用户 ID |
| connected | bool | true=上线，false=下线 |

**使用场景**：多端同步时，一端登录/登出，其他端通过此消息感知在线状态变化。

#### 2.3.5 ListenerMessage — 监听器管理消息

```proto
message ListenerMessage {
    Type type = 1;
    OpType opType = 2;
    string userId = 3;
    string projectId = 4;
    string content = 5;
    uint64 msgPublishAt = 6;

    enum Type {
        UNKNOWN = 0;
        DefaultDispatchChannelListener = 1;    // 默认分发通道（单聊）
        GroupDispatchChannelListener = 2;      // 群组分发通道
        MultiOtherOnlineDispatchChannelListener = 3;  // 多端在线通道
    }
    enum OpType {
        OP_UNKNOWN = 0;
        AddListener = 1;     // 添加监听器（订阅）
        RemoveListener = 2;  // 移除监听器（取消订阅）
    }
}
```

**这是 Pub/Sub 架构的核心**：管理"谁订阅了哪个通道"。

| 字段 | 类型 | 作用 |
|---|---|---|
| type | Type enum | 通道类型：单聊 / 群聊 / 多端在线 |
| opType | OpType enum | 操作类型：添加订阅 / 取消订阅 |
| userId | string | 用户 ID |
| projectId | string | 项目 ID（多租户隔离） |
| content | string | 消息内容（通常是 JSON） |
| msgPublishAt | uint64 | 消息发布时间戳（毫秒） |

**三种通道的含义**：

| 通道 | 含义 | 推送内容 |
|---|---|---|
| DefaultDispatchChannelListener | 默认单聊通道 | 私聊消息、系统通知等 |
| GroupDispatchChannelListener | 群组通道 | 群聊消息、群事件 |
| MultiOtherOnlineDispatchChannelListener | 多端在线通道 | 多端同步事件（如一端发消息其他端同步） |

---

## 三、SubProtocol.proto 详解 ⭐ 核心协议

### 3.1 文件基本信息

| 项 | 值 |
|---|---|
| 文件路径 | `TTDb/Protobuf/SubProtocol.proto` |
| 生成文件 | `TTDb/Classes/Protobuf/SubProtocol.pb.swift` |
| 语法 | proto3 |
| package | `app.meeky` |
| Swift 前缀 | `IM` |
| iOS 端使用情况 | ✅ 核心协议，大量使用 |

### 3.2 为什么定义这个文件

这是 **客户端与服务端 WebSocket 通信的主干协议**。设计成 **类 HTTP 风格**（verb + path + body + headers + status），好处是：
1. 开发者熟悉，上手快
2. 可以复用 HTTP 的概念（状态码、请求路径、请求头）
3. 请求-响应模式通过 `id` 匹配，和 HTTP request-response 一一对应
4. 同时支持服务端主动推送（没有对应请求 id 的消息就是推送）

### 3.3 协议分层结构

```
第1层  WebSocketMessage  (WebSocket 帧包装)
  ├─ REQUEST  → WebSocketRequestMessage (verb/path/body/id/headers)
  └─ RESPONSE → WebSocketResponseMessage (id/status/body/headers)

第2层  Envelope  (业务消息信封，在 body 里面)
  ├─ PRIVATE_CONVERSATION → ConversationMessage (单聊消息)
  ├─ GROUP_CONVERSATION   → ConversationMessage (群聊消息)
  ├─ DATA                → DataMessage (通用 JSON 数据)
  ├─ COMMAND             → CommandMessage (指令消息)
  ├─ COMPLEX             → ComplexMessage (批量消息)
  └─ INSTANT_GROUP       → InstantGroupMessage (临时群消息)
```

### 3.4 各 message 详解

#### 3.4.1 WebSocketMessage — WebSocket 帧外层包装

```proto
message WebSocketMessage {
    Type type = 1;
    WebSocketRequestMessage request = 2;
    WebSocketResponseMessage response = 3;

    enum Type {
        UNKNOWN = 0;
        REQUEST = 1;
        RESPONSE = 2;
    }
}
```

**设计思路**：WebSocket 是双向的，一条消息要么是请求（客户端→服务端），要么是响应（服务端→客户端）。用 type + 两个可选字段实现。

| 字段 | 类型 | 作用 |
|---|---|---|
| type | Type enum | REQUEST=请求，RESPONSE=响应 |
| request | WebSocketRequestMessage | 请求体（type=REQUEST 时有值） |
| response | WebSocketResponseMessage | 响应体（type=RESPONSE 时有值） |

#### 3.4.2 WebSocketRequestMessage — 请求消息

```proto
message WebSocketRequestMessage {
    string verb = 1;                       // HTTP 动词
    string path = 2;                       // 请求路径
    google.protobuf.BytesValue body = 3;   // 请求体（可空）
    uint64 id = 4 [jstype = JS_STRING];    // 请求 ID
    repeated string headers = 5;           // 请求头
}
```

**完全模仿 HTTP 请求**：

| 字段 | 类型 | 对应 HTTP | 作用 |
|---|---|---|---|
| verb | string | HTTP Method | GET / POST / PUT / DELETE 等 |
| path | string | URL Path | 如 `/socket/v1/messages/{convId}` |
| body | BytesValue | Request Body | 二进制请求体（序列化后的 Envelope） |
| id | uint64 | - | 请求唯一 ID，用于匹配响应 |
| headers | repeated string | HTTP Headers | 请求头列表，格式 "key:value" |

**为什么 body 用 BytesValue 而不是 bytes？**

`google.protobuf.BytesValue` 是 Google 官方的 Wrapper 类型，可以表达"有值"和"没有值"两种状态。普通 `bytes` 在 proto3 中默认是空 bytes，无法区分"空 body"和"没有 body"。对于 GET 请求，body 应该是 nil（没有 body），而不是空 bytes。

**为什么 id 加 jstype = JS_STRING？**

JS 的 number 类型是 64 位浮点数，最大安全整数是 2^53。uint64 最大值是 2^64-1，超过了 JS 的安全范围，会丢精度。加 `jstype = JS_STRING` 后，JS 端会把它当字符串处理，避免精度丢失。iOS 端不受影响，还是 uint64。

#### 3.4.3 WebSocketResponseMessage — 响应消息

```proto
message WebSocketResponseMessage {
    uint64 id = 1;                         // 对应请求 ID
    uint32 status = 2;                     // HTTP 状态码
    string message = 3;                    // 状态消息
    google.protobuf.BytesValue body = 4;   // 响应体
    repeated string headers = 5;           // 响应头
}
```

对应 HTTP 响应格式：

| 字段 | 类型 | 对应 HTTP | 作用 |
|---|---|---|---|
| id | uint64 | - | 对应请求的 ID，用于匹配请求-响应 |
| status | uint32 | Status Code | 200=成功，4xx=客户端错误，5xx=服务端错误 |
| message | string | Status Text | 状态描述文本 |
| body | BytesValue | Response Body | 二进制响应体 |
| headers | repeated string | Headers | 响应头列表 |

#### 3.4.4 Envelope — 业务消息信封 ⭐

```proto
message Envelope {
    Type type = 1;
    uint64 timestamp = 2;

    oneof content {
        ConversationMessage conversationMessage = 3;
        DataMessage dataMessage = 4;
        CommandMessage commandMessage = 5;
        ComplexMessage complexMessage = 6;
        InstantGroupMessage instantGroupMessage = 7;
    }

    string id = 8;

    enum Type {
        UNKNOWN = 0;
        PRIVATE_CONVERSATION = 1;   // 单聊会话消息
        GROUP_CONVERSATION = 2;     // 群聊会话消息
        DATA = 3;                   // 通用数据消息
        COMMAND = 4;                // 指令消息
        COMPLEX = 5;                // 复合/批量消息
        INSTANT_GROUP = 6;          // 临时群组消息
    }
}
```

**这是整个 IM 系统最核心的消息结构**。所有业务消息都包装在 Envelope 里，放在 WebSocketRequestMessage / WebSocketResponseMessage 的 body 中传输。

**为什么用 oneof 而不是 type + bytes？**

对比 PubSubMessage 的 type + bytes 设计，这里用了 proto3 的 `oneof` 特性：
- 优点：类型安全，编译期检查，自动反序列化
- 优点：内存里只有一个字段有值，节省空间
- 缺点：新增类型需要修改 Envelope 定义

PubSubMessage 用 type + bytes 是因为服务端转发时不需要解析内容，而 Envelope 是业务层消息，客户端需要解析每个字段，所以用 oneof 更合适。

| 字段 | 类型 | 作用 |
|---|---|---|
| type | Type enum | 消息类型，决定 oneof content 中哪个字段有值 |
| timestamp | uint64 | 消息时间戳（毫秒级） |
| content | oneof | 消息体，5 种类型二选一 |
| id | string | 信封 ID（消息唯一标识） |

#### 3.4.5 ConversationMessage — 聊天消息 ⭐⭐

```proto
message ConversationMessage {
    string id = 1;                    // 消息 ID
    string conversationId = 2;        // 会话 ID
    uint32 type = 3;                  // 消息类型
    string senderId = 4;              // 发送者 ID
    string content = 5;               // 消息内容（JSON）
    string extras = 6;                // 扩展信息
    uint64 createdAt = 7;             // 创建时间
    uint64 updatedAt = 8;             // 更新时间
    uint64 messageCount = 9;          // 消息计数
    bool updateLatestMessage = 10;    // 是否更新会话最新消息
    uint64 senderReadCount = 11;      // 发送方已读数
    repeated string atIds = 12;       // @ 的用户 ID 列表
}
```

**这是聊天消息的核心数据结构**，单聊和群聊都用它。

| 字段 | 类型 | 作用 | 详细说明 |
|---|---|---|---|
| id | string | 消息唯一 ID | 全局唯一，用于去重、撤回、引用等 |
| conversationId | string | 会话 ID | 单聊或群聊的会话标识 |
| type | uint32 | 消息类型 | 0=未知, 1=文本, 2=图片, 3=系统, 4=内部, 5=阅后即焚, 6=指令... 具体值见 ChatType.swift |
| senderId | string | 发送者用户 ID | 谁发的这条消息 |
| content | string | 消息内容（JSON） | 不同 type 有不同 JSON 结构，如文本消息: `{"text":"hello"}` |
| extras | string | 扩展信息（JSON） | 存放额外业务数据，如引用消息、AI 消息元数据等 |
| createdAt | uint64 | 创建时间戳 | 消息发送时间（毫秒） |
| updatedAt | uint64 | 更新时间戳 | 消息最后修改时间（如撤回、编辑） |
| messageCount | uint64 | 消息计数 | 该会话的消息序号 |
| updateLatestMessage | bool | 是否更新会话最新消息 | 有些内部消息不更新会话列表的最新消息预览 |
| senderReadCount | uint64 | 发送方已读数 | 发送方已读了多少条消息，用于已读同步 |
| atIds | repeated string | @ 的用户 ID 列表 | 群聊中 @ 谁，对应消息类型中的 at 功能 |

**为什么 content 和 extras 是 string（JSON）而不是 Protobuf 结构？**

这是一个关键设计决策：
1. **灵活性** — 消息类型（type）有几十种，每种的 content 结构都不同，如果全部用 Protobuf 定义，协议文件会非常臃肿
2. **前后端解耦** — 新增消息类型不需要改 proto 文件重新生成代码，JSON 可以直接解析
3. **历史原因** — 早期可能是纯 JSON 协议，后来迁移到 Protobuf 时保留了 JSON content

**数据库转换**：`ConversationMessage` 不会直接存数据库，会转换成 `IMMessage`（WCDB 表模型）后存入 `msgTable`。转换位置在 `TTDb/Classes/Model/IM/IMMessage.swift` 的 `init(convType:recvMsg:)`。

#### 3.4.6 DataMessage — 通用数据消息

```proto
message DataMessage {
    string content = 1;  // JSON 内容
}
```

最简单的消息类型，只有一个 `content` 字段（JSON 字符串）。

**用途**：
- 拉取历史消息的请求和响应
- 批量获取会话信息
- 各种 HTTP over WebSocket 的 API 调用
- 任何不需要结构化 Protobuf 的数据传输

**为什么叫 DataMessage？** 因为它承载的是"数据"而非"消息"，类似 HTTP 的 body。

#### 3.4.7 CommandMessage — 指令消息

```proto
message CommandMessage {
    Type type = 1;
    string content = 2;  // JSON 内容

    enum Type {
        UNKNOWN = 0;
        CONVERSATION_CREATION = 1;   // 会话创建
        PROJECT_CUSTOMIZED = 2;      // 项目自定义指令
        PRIVATE_MESSAGE_ACK = 3;     // 私聊消息 ACK
        USER_ONLINE_CHANGED = 4;     // 用户在线状态变化
        CONVERSATION_TYPING = 5;     // 正在输入提示
        COMMON_MESSAGE = 6;          // 通用消息
        SOCKET_CONNECTED = 7;        // Socket 已连接
    }
}
```

**指令消息 vs 聊天消息的区别**：
- 聊天消息（ConversationMessage）是用户之间的沟通内容，要存数据库、显示在聊天界面
- 指令消息（CommandMessage）是系统级别的通知，不显示在聊天流里，由代码逻辑处理

| 指令类型 | 触发场景 | 处理方式 |
|---|---|---|
| CONVERSATION_CREATION | 新会话被创建 | 在会话列表中插入新会话 |
| PROJECT_CUSTOMIZED | 项目自定义业务指令 | 交给项目侧自定义处理 |
| PRIVATE_MESSAGE_ACK | 消息已读回执 | 更新消息已读状态 |
| USER_ONLINE_CHANGED | 好友在线状态变化 | 更新在线状态显示 |
| CONVERSATION_TYPING | 对方正在输入 | 显示"对方正在输入..." |
| COMMON_MESSAGE | 通用指令 | 通用处理逻辑 |
| SOCKET_CONNECTED | Socket 连接成功 | 初始化逻辑 |

#### 3.4.8 ComplexMessage — 复合/批量消息

```proto
message ComplexMessage {
    repeated ConversationMessage conversationMessage = 1;
    repeated InstantGroupMessage instantGroupMessage = 2;
    repeated CommandMessage commandMessage = 3;
}
```

**批量投递**：当服务端要一次性推送多条消息时，用 ComplexMessage 打包。

**为什么需要批量？**
- 减少 WebSocket 帧数量，提升性能
- 保证一批消息的原子性（一起到达）
- 比如刚进群时，批量推送未读消息

#### 3.4.9 InstantGroupMessage — 临时群组消息

```proto
message InstantGroupMessage {
    Type type = 1;
    string groupName = 2;
    string senderId = 3;
    string content = 4;
    string senderImUserId = 5;
    string extras = 6;

    enum Type {
        UNKNOWN = 0;
        PROJECT_CUSTOMIZED = 1;
        GROUP_JOINED = 2;
        GROUP_QUIT = 3;
        FORCE_QUIT = 4;
        ITEM_COMPETITION = 5;
        ITEMS_RELEASE = 6;
    }
}
```

**临时群 vs 普通群的区别**：
- 普通群（GROUP_CONVERSATION）有持久化的群 ID、群成员列表，存在数据库里
- 临时群（INSTANT_GROUP）是临时性的，比如一起看片、一起听歌、实时互动等场景，不需要持久化

| 字段 | 类型 | 作用 |
|---|---|---|
| type | Type enum | 临时群事件类型 |
| groupName | string | 群组名称 |
| senderId | string | 发送者 ID |
| content | string | 消息内容（JSON） |
| senderImUserId | string | 发送者 IM 用户 ID |
| extras | string | 扩展信息 |

---

## 四、SubConvertible.proto 详解

### 4.1 文件基本信息

| 项 | 值 |
|---|---|
| 文件路径 | `TTDb/Protobuf/SubConvertible.proto` |
| 生成文件 | `TTDb/Classes/Protobuf/SubConvertible.pb.swift` |
| 语法 | proto3 |
| package | `app.ios` |
| Swift 前缀 | `IM` |
| iOS 端使用情况 | ✅ 历史消息拉取用 |

### 4.2 为什么定义这个文件

`SubConvertible` = **Sub**-Protocol **Convertible** = 子协议可转换类型。

这些是 **业务数据模型**，通过 SubProtocol 中 `DataMessage.content`（JSON 字符串）传递。它们本身不直接走 WebSocket 传输，而是作为 JSON 解析的目标结构。

**为什么 package 是 app.ios？** 因为这是 iOS 侧定义的业务数据模型，和服务端的 app.meeky 区分开。

### 4.3 各 message 详解

#### 4.3.1 V1MessagesGetRes — 历史消息拉取响应

```proto
message V1MessagesGetRes {
    repeated V1MessagesGetData data = 1;
    int32 count = 2;
    string nextPageToken = 3;
    bool hasNext = 4;
}
```

对应 REST API 的分页响应格式。

| 字段 | 类型 | 作用 |
|---|---|---|
| data | repeated V1MessagesGetData | 消息数据列表 |
| count | int32 | 消息总数 |
| nextPageToken | string | 下一页 Token（游标分页） |
| hasNext | bool | 是否有下一页 |

**为什么用 Token 分页而不是页码分页？**

游标分页（Token-based pagination）的优势：
1. 数据实时变化时不会重复或遗漏（消息可能实时插入）
2. 服务端实现简单，不需要 OFFSET（大偏移量性能差）
3. 适合 IM 场景的滚动加载

#### 4.3.2 V1MessagesGetData — 单条历史消息数据

```proto
message V1MessagesGetData {
    string id = 1;
    string conversationId = 2;
    uint32 type = 3;
    string senderUid = 4;
    string content = 5;
    string extras = 6;
    int64 createdAt = 7;
    int64 updatedAt = 8;
    repeated string readUsers = 9;
    bool burn = 10;
    bool revoked = 11;
    repeated string atIds = 12;
}
```

和 `ConversationMessage` 非常相似，这是历史消息 API 返回的消息格式。

| 字段 | 类型 | 与 ConversationMessage 对比 | 作用 |
|---|---|---|---|
| id | string | 相同 | 消息 ID |
| conversationId | string | 相同 | 会话 ID |
| type | uint32 | 相同 | 消息类型 |
| senderUid | string | 对应 senderId | 发送者 UID（字段名不同） |
| content | string | 相同 | 消息内容（JSON） |
| extras | string | 相同 | 扩展信息 |
| createdAt | int64 | 对应 uint64 | 创建时间（类型不同，int64 vs uint64） |
| updatedAt | int64 | 对应 uint64 | 更新时间 |
| readUsers | repeated string | 新增（ConversationMessage 没有） | 已读用户列表 |
| burn | bool | 新增 | 是否阅后即焚 |
| revoked | bool | 新增 | 是否已撤回 |
| atIds | repeated string | 相同 | @ 用户列表 |

**为什么历史消息的字段比实时消息多？**

实时推送的 `ConversationMessage` 只包含必要字段（为了传输效率），而历史消息 API 返回完整的消息数据，包括已读列表、是否阅后即焚、是否撤回等状态字段。

**数据库转换**：`IMV1MessagesGetData` 转换为 `IMMessage` 后存入数据库，转换位置在 `TTDb/Classes/Model/IM/IMMessage.swift` 的 `init(convType:syncMsg:)`。

---

## 五、三个文件的关系与完整调用链路

### 5.1 架构关系图

```
服务端 Pub/Sub 层
  │
  │ PubSubMessage (服务端内部)
  │  ├─ DELIVER → 投递业务消息
  │  ├─ KEEPALIVE → 心跳
  │  └─ INTERNAL → 内部消息(Connected/Listener)
  ▼
WebSocket 网关
  │
  │ 包装为 WebSocketMessage (SubProtocol.proto)
  ▼
客户端接收
  │
  │ 第1层：IMWebSocketMessage (帧格式)
  │  ├─ REQUEST / RESPONSE
  │  └─ body → IMEnvelope
  │
  │ 第2层：IMEnvelope (业务信封)
  │  ├─ .conversationMessage → 聊天消息
  │  ├─ .dataMessage → 通用数据 (JSON)
  │  │   └─ JSON 解析 → SubConvertible.proto 的类型
  │  │       └─ 如 IMV1MessagesGetRes (历史消息)
  │  ├─ .commandMessage → 指令消息
  │  ├─ .complexMessage → 批量消息
  │  └─ .instantGroupMessage → 临时群消息
  ▼
业务层 (TTChat / TTMoment / ...)
```

### 5.2 发送消息完整链路

```
业务层 (IMCenter+Send.swift)
  │
  │ 构造 IMEnvelope
  │   type = .data
  │   dataMessage.content = "{\"text\":\"hello\"}"
  ▼
IMSocketManager.send(verb: "POST", path: "/socket/v1/messages", envelope)
  │
  │ 1. envelope.serializedData() → Data
  │    （将 IMEnvelope 序列化为二进制）
  │
  │ 2. IMSocketMessageFactory.createRequest(
  │      verb: "POST",
  │      path: "/socket/v1/messages",
  │      body: envelopeData,
  │      id: requestId
  │    )
  │    → IMWebSocketMessage {
  │        type = .request
  │        request { verb, path, body, id, headers }
  │      }
  │
  │ 3. message.serializedData() → Data
  │    （将 IMWebSocketMessage 序列化为二进制）
  ▼
WebSocket.write(data)  →  发送到服务器
```

### 5.3 接收消息完整链路

```
WebSocket.didReceive(data)
  │
  │ IMSocketMessage(data: data)
  │   → IMWebSocketMessage(serializedData: data)
  │   （反序列化为 WebSocket 帧）
  ▼
IMSocketManager.onMessage(message)
  │
  │ 1. 判断是响应还是推送
  │    - 有对应请求 id → 响应，回调 single(.success)
  │    - 无对应请求 id → 服务端推送，走 process(envelope)
  │
  │ 2. message.getBody() → IMEnvelope
  │    （从 body.value 反序列化为业务信封）
  ▼
IMSocketManager.process(envelope)
  ├─ type == .privateConversation / .groupConversation
  │   → IMCenter.shared.recvMsgs(envelope)
  │     ├─ 提取 IMConversationMessage 数组
  │     ├─ 转换为 IMMessage（数据库模型）
  │     ├─ 插入 WCDB 数据库
  │     └─ 通知聊天页面刷新 UI
  │
  ├─ type == .command
  │   → IMCenter.shared.recvCmdMsgs(envelope)
  │     ├─ 提取 IMCommandMessage
  │     ├─ 转换为 IMCommand
  │     └─ 分发到对应处理逻辑
  │
  └─ type == .complex
      → 分别提取 conversationMessage + commandMessage
      → 批量处理
```

### 5.4 拉取历史消息完整链路

```
IMSocketMessageKeeper.fetchHistoryMsgsFromServer()
  │
  │ 构造 IMEnvelope(type = .data, dataMessage = IMDataMessage())
  │ GET /socket/v1/messages/{convId}
  ▼
WebSocket 发送请求
  │
  ▼
服务端返回响应
  │
  │ IMEnvelope(type = .data)
  │ envelope.dataMessage.content → JSON 字符串
  ▼
IMV1MessagesGetRes(jsonString: content)
  │
  │ JSON 解析为 Protobuf 模型
  ▼
response.data → [IMV1MessagesGetData]
  │
  │ 逐条转换为 IMMessage
  │ IMMessage(convType: .group, syncMsg: data)
  ▼
TTDb.msgTable.insertOrReplace() → 存入数据库
```

---

## 六、数据库中的角色

Protobuf 类型是 **"网络传输层模型"**，不直接存数据库。数据流：

```
网络层 (Protobuf)                    数据库层 (WCDB)
─────────────────                    ───────────────
IMConversationMessage  ──转换──→  IMMessage ──→  msgTable
IMV1MessagesGetData     ──转换──→  IMMessage ──→  msgTable
IMCommandMessage        ──转换──→  IMCommand ──→  cmdTable
```

**转换位置**：
- `TTDb/Classes/Model/IM/IMMessage.swift`
  - `init(convType:recvMsg:)` — 从 `IMConversationMessage` 转换
  - `init(convType:syncMsg:)` — 从 `IMV1MessagesGetData` 转换
- `TTDb/Classes/Model/IM/IMCommand.swift`
  - `init(msg:)` — 从 `IMCommandMessage` 转换

**为什么不直接存 Protobuf 到数据库？**
1. WCDB 是 ORM 数据库，需要 `TableCodable` 协议的模型
2. 数据库字段和网络字段不完全一致（如 senderId vs senderUid）
3. 数据库需要索引、关联查询等优化，Protobuf 模型不适合
4. 网络模型变化频繁，数据库模型需要稳定

---

## 七、从 0 到 1 手写的实现顺序

如果你要照着这份逻辑从零写一份 IM 通信，建议按以下顺序：

| 阶段 | 内容 | 产出 |
|---|---|---|
| 第1步 | 写 `SubProtocol.proto` 的 WebSocketMessage + Request + Response | 能序列化/反序列化 WebSocket 帧 |
| 第2步 | 写 `Envelope` + `DataMessage` | 能通过 DataMessage 收发 JSON 数据 |
| 第3步 | 封装 WebSocket（Starscream）+ 请求-响应匹配（id 映射） | 能发送请求并收到响应 |
| 第4步 | 写 `ConversationMessage` | 能收发聊天消息 |
| 第5步 | 写 `CommandMessage` | 能处理系统指令 |
| 第6步 | 写 `ComplexMessage` | 支持批量消息 |
| 第7步 | 写 `SubConvertible.proto` | 支持历史消息拉取 |
| 第8步 | 数据库模型转换 + WCDB 存储 | 消息持久化 |
| 第9步 | （可选）写 `PubSubMessage.proto` | 服务端 Pub/Sub 架构接入 |

---

## 八、快速参考表

### 8.1 三个文件对比

| 维度 | PubSubMessage.proto | SubProtocol.proto | SubConvertible.proto |
|---|---|---|---|
| 层级 | 服务端 Pub/Sub 层 | WebSocket 传输层 | 业务数据层 |
| package | app.meeky | app.meeky | app.ios |
| 核心类型 | PubSubMessage | WebSocketMessage + Envelope | V1MessagesGetRes |
| iOS 使用量 | 0 引用 | 大量引用 | 少量引用 |
| 关键字段模式 | type + bytes | oneof content | 标准 REST 响应结构 |

### 8.2 核心消息类型速查

| Swift 类型 | 来源 .proto | 用途 | 关键字段 |
|---|---|---|---|
| IMWebSocketMessage | SubProtocol | WebSocket 帧包装 | type / request / response |
| IMEnvelope | SubProtocol | 业务消息信封 | type / oneof content / id |
| IMConversationMessage | SubProtocol | 聊天消息 | id / conversationId / type / content / senderId |
| IMDataMessage | SubProtocol | 通用数据 | content (JSON) |
| IMCommandMessage | SubProtocol | 指令消息 | type / content |
| IMComplexMessage | SubProtocol | 批量消息 | repeated 数组 |
| IMV1MessagesGetRes | SubConvertible | 历史消息响应 | data / count / nextPageToken |
| IMPubSubMessage | PubSubMessage | Pub/Sub 顶层信封 | type / content |

#### .proto 生成swift

### 第一步：安装编译器（如果还没装）

你需要同时拥有 Google 的 `protoc` 编译器和 Apple 的 Swift 插件 `protoc-gen-swift`。

- **最省事的方法**：直接用 Homebrew 安装，它会一次性帮你搞定这两样东西。

  bash

  ```
  brew install swift-protobuf
  ```

  

### 🛠️ 第二步：执行生成命令

在终端里，进入存放 `.proto` 文件的目录，运行 `protoc` 命令。

假设你的三个文件都在当前目录下，命令如下：

bash

```
protoc --swift_out=. PubSubMessage.proto SubConvertible.proto SubProtocol.proto
```

- **`--swift_out=.`**：指定生成的 `.pb.swift` 文件输出到当前目录（`.`）。
- **文件名**：你可以把三个文件名都列在最后，`protoc` 会一次性全部处理，非常方便。
