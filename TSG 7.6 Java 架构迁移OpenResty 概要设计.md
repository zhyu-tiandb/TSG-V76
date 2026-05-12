
## TrustMore 安全网关架构升级概要设计
           

| 项目   | 内容                                   |
| ---- | ------------------------------------ |
| 文档名称 | TSG 7.6 Java 架构迁移OpenResty 概要设计      |
| 所属产品 | TrustMore 边界安全网关（TSG 7.6）            |
| 版本   | 1.0 合并版                              |
| 创建日期 | 2026-05-07                           |
| 前置文档 | 中宇万通安全网关-技术白皮书.pdf、TSG 7.6 架构迁移需求确认书 |
| 编制   | 田东波（研发部）                             |
| 类型   | 概要设计文档                               |

---

## 1. 设计目标与范围

### 1.1 核心目标

- **引擎替换**：将对外服务引擎从 **Java** 平台迁移到 **OpenResty**（Nginx + LuaJIT）
- **功能等价**：所有现有功能行为不变，用户/管理员无感知
- **性能提升**：并发连接数、SSL 吞吐、握手延迟均优于 Java 版本
- **国密支持**：完整支持 SM2/SM3/SM4 及国密 SSL 协议
- **平滑迁移**：支持现有配置导入，客户端无需升级

### 1.2 系统定位

TSG 7.6 是 TrustMore 安全网关的一次**底层架构迁移**，本质为"换引擎不换功能"。核心改造点在 OpenResty 引擎层，其余层级（管理面、后端服务（含认证授权）、操作系统层）保持不变或仅做接口适配。

### 1.3 改造边界

| 层级                | 改造范围 | 说明                           |
| ----------------- | ---- | ---------------------------- |
| **OpenResty 引擎层** | 全部重写 | Java → OpenResty/Lua，核心迁移对象  |
| **后端服务层**         | 接口适配 | 调用方式变化（内部调用 → HTTP/Lua），协议不变 |
| **管理面**           | 不变   | Web UI、三权分立、配置管理保持原样         |
| **操作系统层**         | 不变   | 网络栈、国密驱动、双机热备保持原样            |
| **客户端**           | 不变   | 用户端逻辑不涉及                     |

---

## 2. 总体架构

### 2.1 系统部署视图

```
[外网用户] → [防火墙] → [TrustMore 安全网关（OpenResty）] → [内网应用]
                               │
                               ├─ 反向代理模式（B/S、C/S）
                               ├─ NC 模式（TCP/UDP 隧道）
                               └─ 管理端（后端 Java/Go 不变）

网关仍位于 DMZ 区，支持主路/旁路部署
```

### 2.2 逻辑架构（OpenResty 内部）

```
┌──────────────────────────────────────────────────────────┐
│                  OpenResty (Nginx + Lua)                 │
├──────────────┬──────────────┬──────────────┬──────────────┤
│   SSL/TLS    │  HTTP/HTTPS  │   Stream     │     Lua      │
│   终结层     │    代理层    │    代理层    │   扩展层     │
│ (ngx_ssl/    │ (proxy_pass, │ (stream      │ (access,     │
│ 国密补丁)    │ rewrite)     │ proxy)       │ log,..)      │
├──────────────┴──────────────┴──────────────┴──────────────┤
│       共享缓存 (lua_shared_dict)                          │
│  会话状态 / CRL 缓存 / 动态策略缓存 / 在线用户            │
└────────────────────────────┬─────────────────────────────┘
                             │
                ┌────────────▼──────────────┐
                │    后端服务（保持不变）    │
                │ - 数据库（用户/配置）      │
                │ - 管理端 API              │
                │ - 第三方认证（LDAP/AD）   │
                └───────────────────────────┘
```

**各层职责：**

- **SSL/TLS 终结层**：处理国际 TLS 和国密 SSL，证书验证、算法协商
- **HTTP/HTTPS 代理层**：反向代理、URL 重写、压缩、统一门户
- **Stream 代理层**：TCP/UDP 隧道（NC 模式）、C/S 应用代理
- **Lua 扩展层**：认证、RBAC、动态角色、SSO 表单代填、日志记录、终端安全检查
- **共享缓存**：存储在线用户 Session、CRL 数据、动态策略规则，跨 worker 进程共享

### 2.3 架构设计决策

| # | 决策点 | 决策 | 依据 |
|---|--------|------|------|
| D1 | SSL 终结放在哪层 | OpenResty Nginx 核心层（原生 SSL） | 性能远优于 Lua 实现 |
| D2 | 国密 SSL 怎么支持 | 国密 OpenSSL/Tongsuo 编译进 Nginx | 必须在 SSL 终结层支持国密 |
| D3 | 认证逻辑放哪层 | Lua 业务层（access_by_lua） | 业务逻辑属于 Lua 层 |
| D4 | 第三方认证处理 | Lua 直连或转发后端（Phase 0 评估） | 取决于库成熟度 |
| D5 | 会话状态管理 | shared_dict + 后端持久化 | 高性能读取 + 持久化 |
| D6 | 配置交互方式 | 管理面写后端 → 后端推送/拉取 OpenResty | 管理面不直接操作 Nginx 配置 |
| D7 | NC 模式实现 | OpenResty stream 模块 + Lua | 原生支持 TCP/UDP 代理 |
| D8 | SSO 表单代填 | Lua body_filter 解析 HTML + 注入 | 必须在响应体过滤层实现 |

---

## 3. 核心模块设计

### 3.1 SSL/TLS 与国密支持

**设计要点：**
- 采用 Tongsuo（阿里开源国密 OpenSSL 分支）或国密版 OpenSSL 编译 Nginx
- 配置支持双证书（RSA + SM2），根据客户端协商选择加密套件
- ssl_certificate_by_lua 阶段进行自定义证书验证（CRL、OCSP、证书链）

**关键配置示例：**
```nginx
server {
    listen 443 ssl;
    ssl_certificate     /certs/sm2_cert.pem;
    ssl_certificate_key /certs/sm2_key.pem;
    ssl_certificate     /certs/rsa_cert.pem;
    ssl_certificate_key /certs/rsa_key.pem;
    
    ssl_ciphers "ECDHE-SM2-SM4-CBC-SM3:ECDHE-RSA-AES128-GCM-SHA256:...";
    ssl_certificate_by_lua_file /lua/cert_verify.lua;
    ssl_session_cache shared:SSL:10m;
}
```

### 3.2 用户认证与 Session 管理

**设计要点：**
- 统一在 access_by_lua 阶段处理认证
- Session 存储在 lua_shared_dict 中，key 为 session_id
- 支持多认证源：本地口令、数字证书、RADIUS/LDAP/AD
- 支持双因子：证书 + 口令等组合验证

**Session 数据结构：**
```lua
{
    user_id = "xxx",
    name = "张三",
    roles = {"admin", "operator"},
    login_time = 1746600000,
    expire_time = 1746603600,
    terminal_id = "mac_xxx",
    auth_method = "cert"
}
```

### 3.3 RBAC 与动态角色匹配

**设计要点：**
- 启动时从数据库加载角色-应用映射表到 shared_dict（定期刷新）
- access_by_lua 中根据用户角色 + 请求 URL 判断访问权限
- 动态角色：基于用户属性（证书 CN、OU、IP 段）匹配规则，运行时计算角色

**规则缓存结构：**
```lua
-- 白名单规则示例
role_rules[role_name] = {
    apps = {"/app1/*", "/app2/api"},
    ips = {"192.168.1.0/24"},
    time_ranges = {"09:00-17:00"}
}
```

### 3.4 终端安全检查与绑定

**设计要点：**
- 客户端登录时上报终端指纹、基线信息（HTTP Header 或独立 API）
- OpenResty 调用后端验证接口判断终端是否绑定、基线是否合格
- 不符合条件则返回 403，携带错误码
- 双网隔离：认证通过后，在响应中下发指令，客户端关闭非认证网卡路由

### 3.5 反向代理模式（B/S 应用）

**设计要点：**
- 使用 proxy_pass 直接转发
- rewrite_by_lua 中进行 URL 重写
- header_filter_by_lua 中修改 Set-Cookie 字段
- 应用地址隐藏：统一门户通过相对路径访问

**示例配置：**
```nginx
location ~ ^/app/(.*)$ {
    rewrite_by_lua '
        local new_uri = "/real/" .. ngx.var[1]
        ngx.var.uri = new_uri
    ';
    proxy_pass http://backend_app;
}
```

### 3.6 NC 模式（TCP/UDP 隧道）

**设计要点：**
- 使用 stream 模块监听 SSL/TCP 端口
- 客户端先建立控制连接进行认证和隧道协商
- preread_by_lua 解析协议头，决定后端目标地址
- 支持网状拓扑（点对点隧道）和星型拓扑（分支→总部）

**配置概览：**
```nginx
stream {
    upstream backend_srv {
        server 10.0.0.10:3389;  # 示例 RDGateway
    }
    server {
        listen 8443 ssl;
        ssl_certificate ...;
        ssl_certificate_by_lua_file /lua/nc_auth.lua;
        preread_by_lua_file /lua/nc_preread.lua;
        proxy_pass backend_srv;
    }
}
```

### 3.7 SSO 表单代填

**设计要点：**
- body_filter_by_lua 中修改响应 body
- 使用 Lua gsub 或正则库在 </form> 前注入隐藏字段
- 仅对配置中需要代填的 URL 处理
- 代填信息从 Session 中获取

**限制：** 复杂 JavaScript 生成的表单可能需要更高级解析，先支持简单 HTML Form，复杂场景推荐证书透传。

### 3.8 统一门户

**设计要点：**
- 用户访问根路径 / 时，由 content_by_lua_file 动态生成门户页面
- 从 Session 获取用户角色，查询数据库得到授权应用列表
- 支持管理员定制 Logo、页面标题（通过后端配置）
- 支持认证后自动跳转到指定应用（302 重定向）

### 3.9 日志审计

**设计要点：**
- log_by_lua 阶段记录访问日志
- 日志格式完全兼容原 Java 版本（字段顺序、分隔符）
- 同时写入本地文件（滚动）和通过 lua-resty-socket 发送 SYSLOG
- 管理操作日志仍由后端记录

**日志字段示例：**
```
timestamp|user|src_ip|dest_ip|dest_port|method|url|status|bytes|duration|terminal_id|...
```

### 3.10 双机热备

**设计要点：**
- 主备两台网关共享虚拟 IP（VRRP）
- 配置使用 rsync/unison 同步（证书、配置文件、Lua 脚本）
- 会话状态同步：备机通过 API 定期从主机拉取 shared_dict 快照；或使用 Redis
- 备机处于 standby 状态，不处理业务流量，但同步配置和状态

---

## 4. 核心模块全景

### 4.1 模块划分

```
引擎层模块（OpenResty）
  M1: SSL/TLS 终结模块
  M2: 反向代理模块
  M3: 认证鉴权模块
  M4: RBAC 策略执行模块
  M5: SSO 单点登录模块
  M6: NC 隧道模块
  M7: 统一门户模块
  M8: 终端安全校验模块
  M9: 日志采集模块
  M10: 会话管理模块

后端模块（保持不变）
  M11: 用户与资源管理模块
  M12: 策略管理模块
  M13: 证书管理模块 (CA/CRL/OCSP)
  M14: 审计日志模块
  M15: 第三方认证代理模块
  M16: 高可用管理模块

管理面模块（保持不变）
  M17: 管理控制台（Web UI）
```

### 4.2 模块职责说明

| 模块 | 职责 | OpenResty 实现方式 |
|------|------|-------------------|
| M1: SSL/TLS 终结 | 处理入站 SSL/TLS 连接，包括国密 SSL | ngx.ssl + 国密 OpenSSL |
| M2: 反向代理 | 将请求转发到后端应用，支持 B/S 和 C/S | proxy_pass + stream |
| M3: 认证鉴权 | 处理身份认证（证书/口令/双因子） | ssl_certificate_by_lua + access_by_lua |
| M4: RBAC 策略执行 | 根据用户角色和策略控制访问权限 | access_by_lua |
| M5: SSO 单点登录 | 表单代填、证书透传、Cookie 跨域 | body_filter_by_lua + header_filter_by_lua |
| M6: NC 隧道 | TCP/UDP 隧道建立与数据转发 | stream + Lua |
| M7: 统一门户 | 门户页面服务、应用列表展示 | Lua 模板渲染 |
| M8: 终端安全校验 | 终端指纹验证、安全基线检查 | access_by_lua + 后端 API |
| M9: 日志采集 | 记录用户访问日志，转发 SysLog | log_by_lua |
| M10: 会话管理 | 在线用户状态、超时注销、拔 KEY 注销 | shared_dict + 后端同步 |

---

## 5. 模块间协作关系

### 5.1 核心协作流程

```
                           用户请求
                              │
                              ▼
                     ┌────────────────┐
                     │ M1: SSL/TLS终结│
                     │ (握手/证书验证)│
                     └───────┬────────┘
                             │
                     ┌───────▼────────┐
                     │ M3: 认证鉴权   │ ←→ M15: 第三方认证
                     │ (身份验证)     │ ←→ M13: 证书管理
                     └───────┬────────┘
                             │
                     ┌───────▼────────┐
                     │ M4: RBAC策略   │ ←→ M12: 策略管理
                     │ (权限判断)     │
                     └───────┬────────┘
                             │
                     ┌───────▼────────┐
                     │ M8: 终端安全   │ ←→ M11: 资源管理
                     │ (终端校验)     │
                     └───────┬────────┘
                             │
                     ┌───────▼────────┐
                     │ M10: 会话管理  │ ←→ shared_dict
                     │ (登录状态)     │
                     └───────┬────────┘
                             │
                  ┌──────────┼──────────┐
                  │          │          │
          ┌───────▼────┐ ┌───▼────┐ ┌──▼──────┐
          │ M2: 反向代理│ │M6: NC  │ │M7: 门户 │
          │ (HTTP转发) │ │隧道    │ │         │
          └───────┬────┘ └───┬────┘ └─────────┘
                  │          │
          ┌───────▼────┐ ┌───▼────┐
          │ M5: SSO代填│ │后端应用 │
          │ (表单注入) │ │         │
          └───────┬────┘ └────────┘
                  │
              ┌───┴─────────┐
              ▼             ▼
          后端应用    ┌──────────────┐
                     │ M9: 日志采集  │
                     └────────┬─────┘
                              │
                              ▼
                        M14: 审计存储
```

---

## 6. 关键流程设计

### 6.1 用户登录流程

```
1. 客户端 → OpenResty：POST /login (证书/口令)
2. ssl_certificate_by_lua（若证书认证）：验证证书链、CRL/OCSP
3. access_by_lua：
   a. 提取凭据，调用认证模块（本地/第三方）
   b. 校验终端指纹（查询绑定表）
   c. 校验终端安全基线（调用后端或本地策略）
   d. 分配角色（静态或动态）
   e. 生成 session_id，存入 shared_dict
4. 返回 Set-Cookie + 门户页面重定向
```

### 6.2 访问受保护应用流程

```
1. 客户端携带 session cookie 请求 /app/xxx
2. access_by_lua：
   a. 验证 session 有效性（shared_dict 中存在且未超时）
   b. 根据用户角色 + 请求 URL 执行 RBAC 检查
   c. 可选：检查时间段、源 IP
3. 若通过：proxy_pass 到后端应用
4. 若需要 SSO 代填：body_filter_by_lua 修改响应
5. log_by_lua：记录访问日志
```

---

## 7. 关键技术选型

| 组件 | 选型 | 版本 | 备注 |
|------|------|------|------|
| OpenResty | openresty | 1.21.4.1+ | 包含 Nginx 1.21+ 和 LuaJIT |
| 国密 SSL | Tongsuo | 8.3.0+ | 替换 OpenResty 默认 OpenSSL |
| Lua 库 | lua-resty-* | 官方模块 | session、lock、socket、redis |
| 第三方认证 | lua-resty-ldap / lua-resty-http | - | 可选，也可走后端 API |
| 日志发送 | lua-resty-socket | - | 发送 SysLog UDP |
| 配置转换工具 | Python 脚本 | - | 解析 Java XML/JSON，生成 Nginx conf + Lua |

---

## 8. 性能目标（SG-5300 示例）

| 指标 | 目标值 |
|------|--------|
| 国密最大新建连接数 | ≥4,500 次/秒 |
| 国密 SSL 事务速率 | ≥8,000 次/秒 |
| 国密最大并发连接数 | ≥50,000 |
| 国密加密吞吐量 | ≥1,200 Mbps |
| SSL 握手 P99 延迟 | ≤ 70% Java |
| 接入用户数 | ≥15,000 |

**性能设计策略**（借鉴白皮书思想）：
- 多 worker 进程：利用多核 CPU，每个 worker 独立事件循环
- 连接池：proxy_pass 自带连接池，复用后端连接
- SSL 会话缓存：ssl_session_cache shared:SSL:10m; 减少握手开销
- 缓冲调优：调整 proxy_buffer_size、proxy_buffers 适应大文件传输
- 事件驱动：全 epoll-only 数据处理，无阻塞高效

---

## 9. 非功能性设计

### 9.1 可靠性设计

- **配置热重载**：nginx -s reload 不重启 master，平滑替换 worker
- **健康检查**：health_check 指令或 Lua 实现主动探测
- **熔断**：可扩展实现后端失败重试、降级策略
- **连接管理**：超时控制、连接限流、队列管理

### 9.2 安全性设计

- **限制请求速率**：limit_req 防暴力破解
- **IP 黑白名单**：access_by_lua 中查询数据库或配置
- **防 SQL 注入**：对用户输入在 Lua 层转义
- **国密合规**：所有国密算法均通过 Tongsuo 的认证

### 9.3 可维护性设计

- **日志分级**：error.log、access.log 按天滚动
- **调试接口**：开放 /_status 返回 shared_dict 统计和 worker 状态
- **CLI 工具**：tsg-cli 用于配置检查、缓存清理、手动同步
- **配置版本管理**：Git 管理所有配置和 Lua 脚本

---

## 10. 配置与迁移设计

### 10.1 配置文件转换

- **原 Java 配置格式**：XML（或 JSON）
- **转换工具**读取原配置文件，生成：
  - nginx.conf：全局配置、监听端口、SSL 设置
  - conf.d/*.conf：各应用代理配置
  - lua/*.lua：认证、授权、SSO 等动态逻辑
  - shared_dict 初始化数据

### 10.2 迁移步骤

1. 备份原 Java 网关配置
2. 运行转换工具，输出 OpenResty 配置目录
3. 复制证书文件到指定目录
4. 启动 OpenResty，验证功能
5. 灰度切换：通过 DNS/负载均衡将部分流量导入新网关

---

## 11. 阶段实施概要

| 阶段 | 内容 | 产出 | 优先级 |
|------|------|------|--------|
| Phase 0 | 国密 PoC 验证 | 可行性报告 | ⭐⭐⭐ |
| Phase 1 | 核心流量迁移 | SSL/TLS + 反向代理 + 认证 + RBAC | ⭐⭐⭐ |
| Phase 4 | 全量功能 + 性能测试 | 测试报告 | ⭐⭐⭐ |
| Phase 2 | 高级功能迁移 | SSO + NC + 双机热备 | ⭐⭐ |
| Phase 3 | 监控审计适配 | 日志 + SNMP + SysLog | ⭐⭐ |
| Phase 5 | 兼容性验证 + 试运行 | 验收报告 | ⭐ |

**建议周期**：Phase 0 (1周) > Phase 1 (4-6周) > Phase 4 (2-3周) > Phase 2 (3-4周) > Phase 3 (2周) > Phase 5 (1-2周)

---

## 12. 风险管理

### 12.1 主要风险点

| 风险 | 等级 | 应对措施 |
|------|------|----------|
| 国密 OpenSSL 与 OpenResty 集成 | 🔴 高 | Phase 0 PoC 验证 |
| SSO 表单代填复杂 | 🟡 中 | 分阶段迁移，优先主流方案 |
| 双机热备状态同步 | 🟡 中 | shared dict + 后端同步 |
| Lua 第三方库不成熟 | 🟡 中 | 后端 API 转发备用方案 |
| 性能未达预期 | 🟢 低 | 保留 Java 版本回退方案 |
| NC 模式隧道协议兼容性 | 🟡 中 | Phase 1 优先验证 |

### 12.2 应急预案

- 性能预测不达：保留 Java 版本作为 fallback
- 国密集成失败：采用纯 RSA 方案先上线，后续追加国密
- 功能缺陷：灰度发布，快速回滚

---

## 13. 有待明确的问题（Open Items）

| # | 问题 | 负责部门 | 优先级 |
|---|------|---------|--------|
| 1 | Tongsuo 与 OpenResty 的编译兼容性验证 | 研发 | 高 |
| 2 | NC 模式隧道协议细节和客户端兼容性 | 研发 | 高 |
| 3 | SSO 表单代填对复杂 JS 表单的支持范围 | 产品+研发 | 中 |
| 4 | 双机热备会话同步采用 pull 还是外部 Redis | 研发 | 中 |
| 5 | 现有客户最大配置规模（应用数、用户数）测试 | 测试 | 中 |
| 6 | 性能测试基线对标 Java 版本 | 测试 | 中 |

---

## 14. 参考文档与附录

### 14.1 参考文档

- 《5G SSL VPN 技术白皮书（终版）》— 提供控制/数据面分离思想
- 《TSG 7.6 架构迁移确认书 v1.0》 — 需求确认
- 《TSG 7.6 需求分析文档（PRD）》 — 产品需求
- OpenResty 官方文档 — https://openresty.org/
- Tongsuo 国密 OpenSSL 使用指南 — https://github.com/Tongsuo-Project/Tongsuo
- Lua 5.1 Reference Manual — 基础语言文档

### 14.2 Java → OpenResty 模块映射表

| Java 类/功能 | OpenResty 实现位置 | Lua 文件示例 |
|----------|------------------|------------|
| SSLServerSocket | nginx.conf ssl 配置 | - |
| X509Certificate 验证 | ssl_certificate_by_lua | cert_verify.lua |
| UsernamePasswordAuthenticator | access_by_lua | auth_local.lua |
| LDAPAuthenticator | access_by_lua | auth_ldap.lua |
| SessionManager | shared_dict + lua-resty-session | session.lua |
| RBACFilter | access_by_lua | rbac.lua |
| TerminalBindFilter | access_by_lua | terminal_check.lua |
| ReverseProxyServlet | proxy_pass | - |
| URLRewriter | rewrite_by_lua | rewrite.lua |
| SSOFormFiller | body_filter_by_lua | sso_form.lua |
| AccessLogFilter | log_by_lua | access_log.lua |
| SystemMonitor（部分） | lua-resty-prometheus | monitor.lua |
| PortalServlet | content_by_lua | portal.lua |

---

## 15. 一句话结论

> **TSG 7.6 架构迁移本质为"换引擎不换功能"**，通过将 Java 服务引擎替换为 OpenResty（Nginx + LuaJIT），实现性能提升、国密支持完善、运维复杂度降低。本概要设计提供清晰的模块划分、核心流程、技术实现点，指导研发、测试和运维团队执行迁移。

---

## 文档审批

| 角色 | 姓名 | 日期 | 签名 |
|------|------|------|------|
| 设计负责人 | 田东波 | 2026-05-08 | ______ |
| 产品经理 | | | ______ |
| 技术评审 | | | ______ |
| 质量部审核 | | | ______ |

**版本历史**

| 版本 | 日期 | 编制人 | 变更说明 |
|------|------|--------|----------|
| 0.1 | 2026-05-07 | 田东波 | 初稿 |
| 1.0 合并版 | 2026-05-08 | 合并 | 合并三份概设，统一格式和内容 |
