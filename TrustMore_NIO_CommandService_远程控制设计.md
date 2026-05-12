# TrustMore → NIO CommandService 远程控制设计

> **文档类型**: 技术设计说明
> **作用**: 说明 TrustMore 中 CommandService 这一层如何用于远程控制 NIO 引擎，以及去掉该层对功能的影响。
> **保存位置**: 当前目录

---

## 1. 结论概览

- TrustMore 中通过 `cn.com.tsg.util.RemoteProxy` 使用 `cn.com.tsg.server.cmdservice.client.CommanCaller` 调用 `CommandService`，实现对 NIO/Proxy 的远程控制。
- 这条控制链不等同于 Redis `engineChangeChannel` 配置同步通道，后者是另一条基于 Pub/Sub 的异步配置下发路径。
- 如果直接去掉 `CommandService` 这一层，主要影响的是：
  - NIO 引擎/Proxy 进程的远程启停
  - 远程即刻执行命令和状态查询
  - 某些动态策略下发与服务配置信息下发
  - 备机/双机同步时的远程命令下发
- 直接去掉后，Redis 配置同步仍可保留，但上述即时控制行为将无法继续使用。

---

## 2. CommandService 在 TrustMore 中的定位

### 2.1 角色说明

`CommandService` 在 TrustMore 中作为一种远程控制通道，主要用于：

- 远程启动/停止 NIO 代理进程（`proxy_app`）
- 远程创建/停止 CMS 服务
- 远程下发动态服务和审计策略
- 远程更新特定运行时配置（如用户绑定 IP、端口映射、默认地址等）
- 远程执行一些即时命令和状态查询

### 2.2 关键类

- `TrustMore/src/cn/com/tsg/util/RemoteProxy.java`
  - 这是 TrustMore 端对 `CommandService` 调用的封装入口。
  - 绝大多数远程控制逻辑都在该类中实现。
- `cn.com.tsg.server.cmdservice.client.CommanCaller`
  - `RemoteProxy` 通过它建立远程调用上下文并触发远程方法。

### 2.3 调用模式

远程调用基本模式如下：

```java
CommanCaller caller = new CommanCaller();
caller.setClient(clientname, clientip, serverip);
caller.setMulticalServerPort(cmsport);
caller.instantStaticInvoke("proxy_app", "start", new Object[]{visual_port, false});
```

- `clientname` 固定为 `proxy_client`
- `clientip` 通常是本机地址或备机地址
- `serverip` 是目标引擎所在机器 IP
- `cmsport` 是目标机器上的 CommandService 监听端口

---

## 3. 远程控制方法设计

### 3.1 启停与生命周期控制

#### 3.1.1 启动 NIO 引擎与 CMS

相关方法：

- `startCMS(String ip, int cmsport, int newcmsport, int engineType)`
  - 根据 `engineType` 决定是否调用 `createNewCommandService_NIO` 或 `createNewCommandService`
  - 这是创建新的 CMS 并间接启动 NIO 引擎的入口

- `RemoteProxy` 还可以通过 `methodStaticInvoke("cn.com.tsg.proxy.app.RemoteStartProxyApp", "createApp", ...)` 来创建 Proxy 应用。
- 启动完成后，`commandService` 还会继续调用 `proxy_config.setProperty`、`proxy_config.addProtocol`、`proxy_app.start` 等远程接口，完成引擎启动与协议配置下发。

#### 3.1.2 停止 NIO 引擎与 CMS

相关方法：

- `stopCMS(String ip, int cmsport, int engineType)`
  - 先调用 `proxy_app.stopApp` 停止 Proxy 应用
  - 再调用 `LvsProcess.stopCommandService` 或 `stopCommandService_NIO` 停止远程 CMS

- 该逻辑是 TrustMore 执行远程关闭引擎和 CommandService 的主要手段。

#### 3.1.3 状态查询

- `getProxyStatus(String ip, int cmsport)`
  - 通过 `proxy_app.isStarted` 查询远程 Proxy 是否已启动
  - 返回 `1` 表示运行，`0` 表示未运行或调用失败

- `getCmsStatus(String ip, int cmsport)`
  - 目前返回固定值 `1`，但该方法的存在说明 CommandService 层预期承担状态获取职责。

---

### 3.2 远程命令执行与配置下发

#### 3.2.1 动态服务下发

相关方法：

- `addDynamicService(Object obj)`
  - 远程调用 `proxy_service.addService`
  - 该方法用于在运行中的 Proxy 上动态添加一个服务策略

- `updateDynamicService(Object source, Object target)`
  - 先远程调用 `proxy_service.removeService(source)`
  - 再调用 `proxy_service.addService(target)`
  - 用于在线更新一个服务策略

- `removeDynamicService(Object obj)`
  - 远程调用 `proxy_service.removeService`
  - 用于删除运行中的服务策略

#### 3.2.2 审计策略下发

相关方法：

- `RemoteProxyControlMessageStatus(boolean status)`
  - 远程调用 `control_message_audit.setStatus`
  - 用于启用/禁用报文审计策略

- `RemoteProxyControlMessageAuditAgreementPort(Map<String, List<String>> portmap)`
  - 远程调用 `control_message_audit.setPortmap`
  - 用于下发各协议的端口审计映射

#### 3.2.3 数据下发：用户绑定 IP / 默认地址

- `RemoteProxySendUserBindIp()`
  - 远程调用 `proxy_service.setUserBindIpMap`
  - 用于将 TrustMore 中的用户绑定 IP 信息推送到远程 Proxy 引擎

- `RemoteProxySendIpManageDefaultIpAddress()`
  - 远程调用 `proxy_service.setDefaultIpAddressMap`
  - 用于下发放管服 IP 管理缺省地址配置

#### 3.2.4 运行时配置与协议模型

在 `startCMS` 或初始化流程中，`RemoteProxy` 还远程调用了：

- `proxy_config.setProperty`（如 `max_connection_count`、`sslUninstall` 等）
- `proxy_config.addProtocol`（协议类型注册）
- `unissl_whitelist.addUrls`（SSL 白名单下发）

这些配置是在远程引擎上即时写入运行时配置对象。

---

### 3.3 备机同步

- 大多数远程调用逻辑不仅对主机执行，还会循环 `getBackupServerIp()`，对备机 IP 逐一重复同样的调用。
- 这意味着 `CommandService` 层承担了主/备机同步的远程命令传递责任。

---

## 4. 典型调用场景

### 4.1 资源管理与应用策略

`RemoteProxy` 相关调用分布在多个模块中，例如：

- `TrustMore/src/cn/com/tsg/admin/action/resource/BsAppAction.java`
- `TrustMore/src/cn/com/tsg/admin/action/resource/CsAppAction.java`
- `TrustMore/src/cn/com/tsg/admin/action/resource/CsMapAction.java`
- `TrustMore/src/cn/com/tsg/admin/action/resource/NvrAppAction.java`
- `TrustMore/src/cn/com/tsg/util/event/strage/StrageEventLinstenerImpl.java`

这些模块使用 `RemoteProxy` 实现动态服务添加/更新/删除，侧重于 `CommandService` 方式的在线、立即下发。

### 4.2 运行与引擎控制

`RemoteProxy` 也被用于：

- `TrustMore/src/cn/com/tsg/admin/action/resource/AgentAction.java`
- `TrustMore/src/cn/com/tsg/admin/action/servicecenter/ServiceCenterAction.java`
- `TrustMore/src/cn/com/tsg/admin/action/sysconf/RecoveryAction.java`

这些场景往往与 NIO/CMS 启停、远程重启、负载均衡策略初始化等运行时控制有关。

---

## 5. 去掉 CommandService 的功能影响

### 5.1 直接失效的功能点

| 功能 | 受影响情况 | 说明 |
|------|-----------|------|
| NIO / Proxy 远程启动 | 直接失效 | `proxy_app.start`、`createNewCommandService_NIO` 依赖 CommandService 通道 |
| NIO / Proxy 远程停止 | 直接失效 | `proxy_app.stopApp`、`stopCommandService[_NIO]` 依赖 CommandService |
| 远程即时状态查询 | 直接失效 | `proxy_app.isStarted`、`getProxyStatus` 依赖 CommandService |
| 动态服务下发 | 直接失效 | `proxy_service.addService` / `removeService` / `setUserBindIpMap` 等需远程命令执行 |
| 审计策略开关/端口下发 | 直接失效 | `control_message_audit.*` 依赖远程调用通道 |
| 备机同步命令 | 直接失效 | 主/备机双向下发目前通过 `CommandService` 完成 |
| 运行时配置写入 | 直接失效 | `proxy_config.setProperty` / `addProtocol` / `unissl_whitelist.addUrls` 等命令无法下发 |

### 5.2 间接削弱的功能点

- 远程 `iptables` / 访问控制规则可能也会受到影响，因为部分 `RemoteProxy` 方法会依赖 CommandService 或远程命令执行
- 系统维护工具（如远程激活脚本、开关机命令）会失去可控路径
- 若 TrustMore 依赖该通道在后台快速修复或重启 F5/keepalived 等逻辑，则可用性降低

### 5.3 不受影响的功能点

- Redis `engineChangeChannel` 的配置下发路径
  - 该路径仍然可以继续工作
  - 但它只覆盖 Redis 消息支持的 `changetype` 事件，不能替代所有 `CommandService` 下发的即时命令
- 纯数据库写入逻辑
- TrustMore UI 的配置保存/DB 更新逻辑

---

## 6. 设计建议

### 6.1 如果想拆除 CommandService

- 必须定义替代通道：REST/RPC/Redis/消息队列等
- 必须保证以下语义可替代：
  - 引擎启动/停止命令
  - 运行时配置写入
  - 动态策略服务下发
  - 用户绑定 IP、默认地址、端口映射等即时推送
  - 主备机同步

### 6.2 推荐逐步迁移策略

1. 先保留 `CommandService`，将远程控制接口抽象成统一服务层
2. 将关键控制命令迁移到可替代通道（例如：直接 HTTP 管理接口或增强 Redis 指令格式）
3. 逐条验证 `proxy_service.*`、`proxy_app.*` 和 `control_message_audit.*` 的功能等价性
4. 最终关闭 `CommandService` 之前，保证 `getProxyStatus` / `isStarted` / `stopApp` 等核心命令已有可靠替代方案

---

## 7. 结论

`CommandService` 在 TrustMore 中承担的是“远程即时控制”与“运行时命令下发”职责，与 Redis `engineChangeChannel` 的“异步配置同步”职责不同。

去掉 `CommandService` 这一层，不会直接破坏 Redis 配置同步，但会影响 NIO 引擎的远程启停、即时策略下发、状态查询与主备同步等关键控制功能。

## 8. IP / 网关 / Linux 命令脚本使用结论

- TrustMore 中 IP 绑定、网关配置、iptables 规则、keepalive 状态查询、激活脚本执行等，确实也采用了 `RemoteProxy` + `CommanCaller` / `CommandService` 的远程执行方式。
- 这些功能在 `RemoteProxy.java` 中至少包含 6 个核心方法：`setIptablesRule`、`setDynamicGatewayModel`、`getIptablesFlow`、`executeActivationScript`、`getLinuxKeepaliveStatus`、`linuxPingIpaddress`。
- 该类还通过 `proxy_service.setUserBindIpMap`、`proxy_service.setDefaultIpAddressMap` 等接口下发用户绑定 IP、默认地址等网络配置。
- 因此，`CommandService` 这一层不仅仅用于 NIO 启停和策略下发，也覆盖了网络层面和运维层面的远程 Linux 命令执行。
- 如果去掉 `CommandService`，这些基于远程命令执行的 IP/网关配置和 Linux 脚本操作也需要找到替代通道，否则会丢失现有控制能力。
