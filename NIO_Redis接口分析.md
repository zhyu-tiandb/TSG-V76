# NIO 项目 Redis 接口分析

## 一、Redis 通讯接口概览

NIO 项目中与 Redis 通讯的接口主要有 **4 处**：

---

## 二、Redis 接口详解

### 1. JedisConn - Redis 连接工具类

**位置**: `NIO/util/src/cn/com/tsg/util/redis/JedisConn.java`

**作用**:
- 管理 Redis 连接池 (`JedisPool`)
- 提供发布/订阅功能 (`publish`, `subscribe`)
- 提供 KV 查询功能 (`getMessage`)

**调用时机**:
- **认证阶段**: 在 `TokenIdBasedAuthenticator.java:53` 中，通过 `getMessage("token_" + tokenId)` 从 Redis 获取用户认证信息
- **缓存检查**: 在 `GlobalObjects.java:64,77` 中，定时检查用户绑定缓存的有效性

---

### 2. SessionJedisPubSubImpl - Session 过期监听

**位置**: `NIO/proxy-api/src/cn/com/tsg/redis/SessionJedisPubSubImpl.java`

**作用**:
- 继承 `JedisPubSub`，订阅 Redis Key 事件
- 监听 `__keyevent@0__:expired` (超时) 和 `__keyevent@0__:del` (注销) 事件

**调用时机**:
- **Session 失效时**: 当 Redis 中的 Session Key 过期或被删除时，触发 `SessionList.redisInvalid()` 主动注销本地会话

---

### 3. RedisSubscribe/RedisMsgPubSubListener - 配置变更通道

**位置**:
- `NIO/proxy-api/src/cn/com/tsg/redis/newhandle/RedisSubscribe.java`
- `NIO/proxy-api/src/cn/com/tsg/redis/newhandle/RedisMsgPubSubListener.java`

**作用**:
- 订阅 `engineChangeChannel` 通道
- 接收配置下发消息（引擎策略、协议信息、证书透传等）

**调用时机**:
- **系统初始化**: `RedisHandleUtils.initRedisChannel()` 启动监听线程
- **配置变更**: 管理中心下发配置时，通过 `ChannelMessageHandleControl` 分发处理

---

### 4. TsgRedisClient/RedisClientHandler - Netty Redis 客户端

**位置**: `NIO/proxy-api/src/cn/com/tsg/redis/TsgRedisClient.java`

**作用**:
- 基于 Netty 的异步 Redis 客户端
- 支持 AUTH 认证和命令发送
- 提供 `expire()` 方法设置 Key 过期时间

**调用时机**:
- 在 `AuthSessionUpdator` 中用于批量更新会话过期时间

---

## 三、数据处理流程与 Redis 调用时机

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          NIO 代理数据处理流程                                  │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐
│  ① 客户端发起请求  │  SSL/TLS 握手 + 应用数据
└────────┬─────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ② FrontendStage.initChannel() - 前端连接初始化                                 │
│    - WorkerConnReleaseHandler (连接数限制)                                    │
│    - FrontendEventHandler (连接事件)                                          │
│    - [SSL] GbSslHandler/SslHandler ←━━━━━━━━━┐                               │
│    - SslEventHandler (SSL事件)                 │                              │
│    - PeekFrameHandler (协议探测)               │  SSL 解密在这里完成            │
│    - VpnPipelineBuilder (动态构建Pipeline)     ─┘                              │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ③ VpnPipelineBuilder.channelRead() - 协议识别与Pipeline构建                   │
│    ├─ 判断协议类型 (BS=HTTP / CS=自定义协议)                                   │
│    ├─ BS协议: HttpRequestDecoder → HttpResponseEncoder → Compressor         │
│    └─ CS协议: CsVpnRequestDecoder → CsCommandEncoder                         │
│                                                                              │
│    创建 ProxyEngine 并设置:                                                   │
│      - IAuthenticator (认证器: BsAuthenticator/CsAuthenticator)              │
│      - IAuthorization (授权器)                                                │
│      - IIoModel (IO模型)                                                      │
│      - IResponder (响应器)                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ④ RequestEventDispatcher.channelRead() - 请求分发                             │
│    - 创建 IOEvent(msg, isFrontend=true)                                       │
│    - 调用 engine.run(event)                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ⑤ ProxyEngine.run() - 引擎状态机处理                                          │
│    状态: REQUEST_RECEIVED → AUTHENTICATING → AUTHORIZATION_OK → PROXY_RELAY  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ⑥ 【Redis调用点1】TokenIdBasedAuthenticator.authentication() - 认证          │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │  JedisConn.getPublishInstance().getMessage("token_" + tokenId)      │  │
│    │         │                                                          │  │
│    │         ▼                                                          │  │
│    │  ┌─────────────────┐     从 Redis 获取用户认证信息                   │  │
│    │  │   Redis Server   │◀─────────────────────────────────────────────│  │
│    │  │  Key: token_xxx  │     JSON格式:                                │  │
│    │  │  Value: {       │     {"userinfo":{                            │  │
│    │  │    "username":..│       "username", "name", "id",              │  │
│    │  │    "cert":{...}│       "cert":{...}, "sessionTimeout",         │  │
│    │  │    "tsg_session"│       "tsg_session"                          │  │
│    │  │  }}             │     }}                                       │  │
│    │  └─────────────────┘                                                │  │
│    │         │                                                          │  │
│    │         ▼                                                          │  │
│    │  AuthenticationRedisResultCallback.onSuccess()                    │  │
│    │    - AuthApiUtil.parseJsonAuthenticationResult() 解析用户信息       │  │
│    │    - 创建 User 对象 (包含 username, cert, authSessionId, timeout)  │  │
│    │         │                                                          │  │
│    │         ▼                                                          │  │
│    │  SessionList.createSession() - 创建本地会话                          │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│    认证成功 → 状态: AUTHENTICATION_OK                                         │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ⑦ ProxyEngine.authorization() - 授权                                         │
│    - BsAuthorization/CsAuthorization.authorization()                          │
│    - 检查用户访问权限                                                          │
│    授权成功 → 状态: AUTHORIZATION_OK                                          │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ⑧ ProxyEngine.connectToBackend() - 连接后端服务器                             │
│    - ioModel.connectToBackend()                                               │
│    - BackendHanlder.initChannel() 初始化后端Pipeline                          │
│    - [如需要] 后端 SSL: GbSslHandler (客户端模式)                              │
│    连接成功 → 状态: PROXY_RELAY                                               │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ⑨ ProxyEngine.relay() + requestFilter() - 请求过滤与转发                      │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │  遍历 requestFilters:                                                │  │
│    │    1. SessionFilter - 更新会话访问时间                                 │  │
│    │       │                                                               │  │
│    │       ▼                                                               │  │
│    │  【Redis调用点2】SessionFilter.filter()                                │  │
│    │  ┌────────────────────────────────────────────────────────────────┐ │  │
│    │  │  if (redisSession) {                                           │ │  │
│    │  │      AuthUtil.updateAuthSession(session);                      │ │  │
│    │  │          │                                                    │ │  │
│    │  │          ▼                                                    │ │  │
│    │  │  AuthSessionUpdator.update()                                 │ │  │
│    │  │      - 加入 accessedSessions 集合                             │ │  │
│    │  │      - 下次 flush 时批量更新 Redis                             │ │  │
│    │  │  }                                                           │ │  │
│    │  └────────────────────────────────────────────────────────────────┘ │  │
│    │                                                                       │  │
│    │    2. 其他业务过滤器 (重写、压缩等)                                      │  │
│    │                                                                       │  │
│    │  ioModel.send2Backend() → 转发到后端服务器                               │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ⑩ 后端服务器处理请求，返回响应                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ ⑪ ProxyEngine.relay() + responseFilter() - 响应过滤与返回                     │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │  遍历 responseFilters:                                               │  │
│    │    - 响应重写、解压缩、头部修改等                                       │  │
│    │                                                                       │  │
│    │  ioModel.send2Frontend() → 返回给客户端                                │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────┐
│ ⑫ 客户端收到响应    │
└──────────────────┘
```

---

## 四、后台异步 Redis 调用

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        后台异步 Redis 交互                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  【Redis调用点3】AuthSessionUpdator.flush() - 定时批量更新会话                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  SessionList 清理线程 (每 session_clean_interval 分钟执行一次)       │   │
│  │      │                                                               │   │
│  │      ▼                                                               │   │
│  │  AuthSessionUpdator.flush()                                         │   │
│  │      │                                                               │   │
│  │      ▼                                                               │   │
│  │  ┌─────────────────┐     TsgRedisClient.expire()                    │   │
│  │  │   Redis Server   │◀──── 更新所有活跃会话的过期时间                 │   │
│  │  │  EXPIRE命令      │     命令: EXPIRE {authSessionId} {timeout}     │   │
│  │  └─────────────────┘                                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  【Redis调用点4】SessionJedisPubSubImpl.onPMessage() - Session过期监听        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  后台订阅线程 (系统启动时创建)                                         │   │
│  │      │                                                               │   │
│  │      ▼                                                               │   │
│  │  订阅 Redis Key事件:                                                  │   │
│  │    - __keyevent@0__:expired  (Key过期)                               │   │
│  │    - __keyevent@0__:del       (Key被删除)                            │   │
│  │      │                                                               │   │
│  │      ▼ Redis 通知事件                                                 │   │
│  │  SessionJedisPubSubImpl.onPMessage(pattern, channel, message)       │   │
│  │      │                                                               │   │
│  │      ▼                                                               │   │
│  │  SessionList.redisInvalid(authSessionId)                            │   │
│  │      - 查找本地对应的 ProxySession                                    │   │
│  │      - 调用 invalid(sessionId) 注销本地会话                            │   │
│  │      - 触发 SessionListener.onInvalide()                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  【Redis调用点5】RedisMsgPubSubListener.onMessage() - 配置变更监听            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  RedisSubscribe.subChannel() - 订阅 engineChangeChannel             │   │
│  │      │                                                               │   │
│  │      ▼ 管理中心下发配置                                                │   │
│  │  RedisMsgPubSubListener.onMessage(channel, message)                 │   │
│  │      │                                                               │   │
│  │      ▼                                                               │   │
│  │  ChannelMessageHandleControl.receivedMessage()                      │   │
│  │      - TokenMessageHandleImpl: Token删除通知 → kill session/连接      │   │
│  │      - 其他配置变更处理                                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  【Redis调用点6】GlobalObjects定时检查 - 定时检查缓存                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  定时任务检查用户绑定缓存是否有效                                       │   │
│  │      │                                                               │   │
│  │      ▼                                                               │   │
│  │  JedisConn.getMessage(key)                                           │   │
│  │      - 如果返回 null，说明 Redis 中已删除，清理本地缓存               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 五、Redis 调用时机总结

| 序号 | 调用点 | 触发时机 | 操作 | Key模式 |
|-----|-------|---------|------|---------|
| 1 | **TokenIdBasedAuthenticator** | 每次需要认证的请求 | GET 获取用户信息 | `token_{tokenId}` |
| 2 | **SessionFilter** | 每次请求 (认证后) | 标记会话需要更新 | - |
| 3 | **AuthSessionUpdator.flush()** | 定时 (默认5分钟) | EXPIRE 批量更新超时 | `{authSessionId}` |
| 4 | **SessionJedisPubSubImpl** | Redis Key事件回调 | 接收过期/删除通知 | `__keyevent@0__:*` |
| 5 | **RedisMsgPubSubListener** | Redis 发布消息 | 配置变更通知 | `engineChangeChannel` |
| 6 | **GlobalObjects定时检查** | 定时检查缓存有效性 | GET 验证Key存在性 | 用户绑定相关Key |

---

## 六、关键 Redis Key 定义

| Key 常量 | 用途 |
|---------|------|
| `token_{tokenId}` | 存储用户认证信息 (JSON格式) |
| `{authSessionId}` | 认证会话ID，用于过期时间管理 |
| `serviceindex` | 所有应用的 ID |
| `cert_passthrough` | 证书透传配置 |
| `proxy_config_setProperty` | 引擎策略 |
| `proxy_config_addProtocol` | 协议信息 |
| `control_message_audit_status` | 报文审计状态 |
| `proxy_service_fgfuser_bind_ip` | 用户绑定 IP |
| `engineChangeChannel` | 配置变更发布/订阅通道 |

---

## 七、相关文件路径

```
NIO/
├── proxy-api/src/cn/com/tsg/
│   ├── redis/
│   │   ├── TsgRedisClient.java              # Netty Redis 客户端
│   │   ├── RedisClientHandler.java          # Redis 客户端处理器
│   │   ├── SessionJedisPubSubImpl.java      # Session 过期监听
│   │   └── newhandle/
│   │       ├── RedisHandleUtils.java        # Redis 工具类
│   │       ├── RedisSubscribe.java          # Redis 订阅
│   │       ├── RedisMsgPubSubListener.java  # 消息监听器
│   │       ├── channelMessageHandle/
│   │       │   └── ChannelMessageHandleControl.java
│   │       └── model/
│   │           ├── RedisChannelEnum.java
│   │           └── RedisSendTypeEnum.java
│   └── proxy/
│       ├── auth/
│       │   ├── TokenIdBasedAuthenticator.java      # Token 认证
│       │   ├── SessionList.java                    # Session 管理
│       │   ├── AuthSessionUpdator.java             # Session 更新器
│       │   └── SessionFilter.java                  # Session 过滤器
│       └── config/
│           └── GlobalObjects.java                  # 全局对象
├── proxy-app/src/cn/com/tsg/proxy/
│   ├── app/
│   │   ├── FrontendStage.java             # 前端连接初始化
│   │   ├── VpnPipelineBuilder.java        # Pipeline 构建
│   │   ├── BackendHanlder.java            # 后端处理器
│   │   └── RequestEventDispatcher.java    # 请求分发
│   └── auth/
│       └── AuthenticationRedisResultCallback.java  # 认证回调
└── util/src/cn/com/tsg/util/redis/
    └── JedisConn.java                     # Jedis 连接工具
```
