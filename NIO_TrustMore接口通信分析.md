# NIO 与 TrustMore 接口通信分析

## 一、接口通信概览

```
┌─────────────────┐                    ┌─────────────────┐
│   NIO 项目      │                    │  TrustMore项目   │
│  (代理引擎)      │                    │  (认证服务)       │
└────────┬────────┘                    └────────┬────────┘
         │                                       │
         │  ┌──────────────────────────────┐    │
         │  │ HTTP 认证接口调用              │    │
         │  └──────────────────────────────┘    │
         ▼                                       ▼
```

---

## 二、通信接口列表

| 序号 | NIO 调用方 | TrustMore 接口 | URL配置项 | 触发时机 |
|-----|-----------|---------------|----------|---------|
| 1 | **WebCertAuthenticator** | `api/authCert.do` | `cert_auth_url` | 用户证书认证（双向SSL） |
| 2 | **AuthUserLogout** | `api/logout.do` | `logout_url` | 用户主动注销/Session超时 |
| 3 | **TsgAuthorizator** | ~~授权接口~~ | ~~`authorization_url`~~ | 当前已改用Redis（代码已注释） |
| 4 | **TokenIdBasedAuthenticator** | ~~Token认证接口~~ | ~~`tokenid_auth_url`~~ | 当前已改用Redis（代码已注释） |

---

## 三、接口调用时机详解

### 3.1 证书认证接口 (authCert.do)

**NIO 调用**: `NIO/proxy-app/src/cn/com/tsg/proxy/auth/WebCertAuthenticator.java:68-88`

```java
// NIO 发起请求
String certVerifyUrl = config.getProperty("cert_auth_url");
String url = "http://" + authIp + ":" + authPort + certVerifyUrl;
url += "?cert=" + certBase64 + "&userIp=" + userIp;
Result<FullHttpResponse> httpResult = NettyHttpClient.executeGet(authRequest, authIp, authPort, ...);
```

**TrustMore 处理**: `TrustMore/src/cn/com/tsg/user/action/ApiAction.java:1064`

```
调用时机:
┌─────────────────────────────────────────────────────────────────┐
│ ① 客户端发起 HTTPS 连接 (双向SSL认证)                             │
│    - SSL握手，NIO 获取客户端证书                                   │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ ② VpnPipelineBuilder 识别为双向SSL请求                             │
│    - biSsl = sslEngine.getNeedClientAuth()                        │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ ③ 【HTTP调用】WebCertAuthenticator.authentication()             │
│    - 构造证书认证请求: GET http://{authIp}:{port}/api/authCert.do │
│    - 参数: cert=Base64编码的证书, userIp=客户端IP                │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ ④ TrustMore ApiAction.authCert() 处理                            │
│    - 解析证书 DN                                                  │
│    - 验证证书有效性 (CRL/OCSP)                                    │
│    - 查询/创建用户账户                                            │
│    - 返回认证结果 XML                                             │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ ⑤ NIO 解析认证结果，创建 Session                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 3.2 注销接口 (logout.do)

**NIO 调用**: `NIO/proxy-api/src/cn/com/tsg/proxy/auth/AuthUserLogout.java:37-44`

```java
// NIO 发起请求
String logoutUrl = config.getProperty("logout_url");
String fullUrl = "http://" + authIp + ":" + authPort + logoutUrl +
                  "?tokenId=" + accessId + "&p=" + port + "&redirect=0";
Result<FullHttpResponse> result = NettyHttpClient.nioHttpGet(logoutRequest, authIp, authPort, ...);
```

**TrustMore 处理**: `TrustMore/src/cn/com/tsg/user/action/ApiAction.java:527`

```
调用时机:
┌─────────────────────────────────────────────────────────────────┐
│ ① 用户主动注销或 Session 超时                                     │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ ② 【HTTP调用】AuthUserLogout.logout()                            │
│    - 构造注销请求: GET http://{authIp}:{port}/api/logout.do      │
│    - 参数: tokenId, p(端口), redirect                            │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ ③ TrustMore ApiAction.logout() 处理                             │
│    - 清理 Session                                                │
│    - 清理 Redis 中的 token 信息                                   │
│    - 返回注销结果                                                 │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ ④ NIO 清理本地会话                                                │
└─────────────────────────────────────────────────────────────────┘
```

---

### 3.3 授权接口 (已废弃，改用 Redis)

**原 NIO 调用**: `NIO/proxy-api/src/cn/com/tsg/proxy/auth/TsgAuthorizator.java:42-53` (已注释)

```java
// 旧方式：HTTP 调用授权接口 (代码已注释)
String url = "http://" + authIp + ":" + authPort + authorization_url +
              "?userId=" + user.getId() + "&serviceId=" + service.getId();
```

**新方式**: `NIO/proxy-api/src/cn/com/tsg/proxy/auth/TsgAuthorizator.java:89-122`

```java
// 新方式：从 Redis 获取授权信息
String rediskey = "token_" + tokenId;
String message = JedisConn.getPublishInstance().getMessage(rediskey);
// 从 message 中解析应用策略 XML
```

---

### 3.4 Token 认证接口 (已废弃，改用 Redis)

**原 NIO 调用**: `NIO/proxy-app/src/cn/com/tsg/proxy/auth/TokenIdBasedAuthenticator.java:70-79` (已注释)

```java
// 旧方式：HTTP 调用 Token 认证接口 (代码已注释)
String url = "http://" + authIp + ":" + authPort + tokenid_auth_url +
              "?tokenId=" + tokenId;
```

**新方式**: `NIO/proxy-app/src/cn/com/tsg/proxy/auth/TokenIdBasedAuthenticator.java:50-63`

```java
// 新方式：从 Redis 获取用户认证信息
String rediskey = "token_" + tokenId;
String message = JedisConn.getPublishInstance().getMessage(rediskey);
```

---

## 四、配置参数

```properties
# 认证服务器配置 (ProxyConfig)
auth_server.ip = 127.0.0.1
auth_server.port = 80

# API 接口配置
cert_auth_url = /api/authCert.do           # 证书认证接口 (仍在使用)
logout_url = /api/logout.do                # 注销接口 (仍在使用)
authorization_url = /api/authorization.do  # 授权接口 (已废弃，改用Redis)
tokenid_auth_url = /api/authTokenId.do     # Token认证接口 (已废弃，改用Redis)
```

---

## 五、数据格式示例

### 5.1 证书认证请求

```
GET http://127.0.0.1:80/api/authCert.do?cert=MIID...&userIp=192.168.1.100
```

### 5.2 证书认证响应 (XML)

```xml
<AuthResult>
    <code>100</code>
    <message>成功</message>
    <id>5710</id>
    <userName>CN=user,O=org</userName>
    <realName>张三</realName>
    <loginType>4</loginType>
    <sessionTimeout>30</sessionTimeout>
    <cert>MIID...</cert>
    <tokenId>abc123...</tokenId>
    <tsg_session>sess456...</tsg_session>
</AuthResult>
```

### 5.3 注销请求

```
GET http://127.0.0.1:80/api/logout.do?tokenId=abc123&p=443&redirect=0
```

### 5.4 注销响应 (XML)

```xml
<AuthResult>
    <code>100</code>
    <message>成功</message>
</AuthResult>
```

---

## 六、相关文件路径

```
NIO/
├── proxy-api/src/cn/com/tsg/proxy/
│   ├── auth/
│   │   ├── AuthUserLogout.java           # 注销接口调用
│   │   ├── TsgAuthorizator.java          # 授权处理 (改用Redis)
│   │   └── AuthenticationResultCallback.java
│   └── http/NettyHttpClient.java         # HTTP 客户端
├── proxy-app/src/cn/com/tsg/proxy/
│   ├── auth/
│   │   ├── WebCertAuthenticator.java     # 证书认证接口调用
│   │   ├── TokenIdBasedAuthenticator.java # Token认证 (改用Redis)
│   │   └── BsAuthenticator.java          # BS协议认证基类
│   └── app/
│       ├── VpnPipelineBuilder.java       # 协议识别与Pipeline构建
│       └── FrontendStage.java            # 前端连接初始化

TrustMore/
└── src/cn/com/tsg/
    ├── user/action/
    │   └── ApiAction.java                # 认证API处理
    │       - authCert()                  # 证书认证处理
    │       - logout()                    # 注销处理
    │       - auth()                      # 通用认证处理
    └── auth/
        └── IndexAction.java              # 首页跳转
```

---

## 七、总结

| 接口类型 | 当前状态 | 说明 |
|---------|---------|------|
| **证书认证** | ✅ 使用 HTTP | NIO → TrustMore，双向SSL场景 |
| **用户注销** | ✅ 使用 HTTP | NIO → TrustMore，主动/超时注销 |
| **Token认证** | 🔄 改用 Redis | 从 HTTP 改为 Redis GET `token_xxx` |
| **用户授权** | 🔄 改用 Redis | 从 HTTP 改为从 Redis 获取应用策略 |

### 架构演进说明

系统从 **HTTP 接口调用** 模式演进为 **Redis 共享数据** 模式：

1. **旧架构**: NIO 每次认证/授权都通过 HTTP 调用 TrustMore 接口
2. **新架构**: TrustMore 将用户信息写入 Redis，NIO 从 Redis 直接读取

**优势**:
- 减少网络调用开销
- 降低 TrustMore 服务器压力
- 提高认证/授权响应速度

**仍保留 HTTP 接口的场景**:
- 证书认证: 需要实时验证证书有效性
- 用户注销: 需要同步清理服务端 Session
