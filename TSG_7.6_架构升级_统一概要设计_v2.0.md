# TSG 7.6 架构升级统一概要设计

**版本**：v2.0（合并版）  
**日期**：2026-05-07  
**作者**：架构设计团队  
**审核**：产品部 / 研发部 / 测试部

---

## 文档说明

本文档基于以下三个初版设计文档合并、对比、优化而成：
- `TrustMore_安全网关架构升级-deepseek版.md`（深度详细版）
- `tsg_7_6_概要设计.md`（精简概览版）
- `TSG 7.6 概要设计_v1.0.md`（结构化版）
- `TrustMore_架构升级_OpenResty_PRD.md`（需求文档版）

合并逻辑：
- 从 **深度详细版** 吸取技术细节和模块设计
- 从 **精简版** 吸取性能指标和阶段规划
- 从 **结构化版** 吸取模块职责清晰的架构划分
- 从 **PRD 版** 吸取需求、范围、验收标准

**优化方向**：避免重复、层次清晰、便于执行

---

## 1. 设计总览

| 项目 | 内容 |
|------|------|
| 项目名称 | TrustMore 安全网关 (TSG 7.6) 架构升级 |
| 改造类型 | 对外服务引擎迁移 |
| 技术方向 | Java → OpenResty (Nginx + Lua) |
| 功能目标 | 功能完全等价、性能显著提升、客户端无感知 |
| 预计周期 | 6-7 个月（Phase 0 ~ Phase 5） |
| 风险等级 | 中等（主要风险：国密 SSL、NC 隧道兼容性） |

---

## 2. 改造背景与目标

### 2.1 现状问题

**Java 版本存在的瓶颈**：
- SSL 握手延迟较高，特别是在高并发场景
- 内存占用大（JVM 开销 + 大量线程占用）
- 并发处理能力受限于线程模型
- 部署和性能调优复杂，学习曲线陡

**客户需求**：
- 支持更高并发（客户数规模不断增长）
- 降低硬件成本（相同性能需要更少资源）
- 简化运维复杂度

### 2.2 设计目标

| 目标 | 具体要求 |
|------|---------|
| **功能等价** | 所有现有功能完整保留，用户/管理员无感知 |
| **性能提升** | 同等硬件条件下，各项指标不低于原版本；内存占用 ≤70% |
| **平滑迁移** | 现有配置、证书、数据可无缝转移；客户端无需升级 |
| **降低成本** | 简化部署、减少资源占用、降低运维门槛 |

### 2.3 改造边界

#### 改造范围（需重写）
| 模块 | 原技术 | 新技术 | 说明 |
|------|--------|--------|------|
| SSL/TLS 终结 | Java SSLContext | OpenSSL + 国密补丁 | 编译进 Nginx，保留国密 SSL |
| HTTP 代理 | Servlet + Filter | proxy_pass + rewrite_by_lua | 原生 Nginx + Lua 扩展 |
| 反向代理 | 自定义代理 | proxy_pass + header 改写 | 使用 Nginx 原生模块 |
| TCP/UDP 隧道 | 自定义 Socket | stream 模块 + Lua | Nginx stream 核心实现 |
| 认证鉴权 | Java 业务代码 | access_by_lua | Lua 执行，调用后端 API |
| RBAC | 内存规则匹配 | Lua 脚本 + shared_dict | 规则缓存在共享内存 |
| SSO 代填 | body 拦截器 | body_filter_by_lua | Lua 解析 HTML 并注入 |
| 日志审计 | Log4j | log_by_lua | 写文件 + SYSLOG 转发 |

#### 不变范围
- 管理后台（Web UI）
- 用户/角色/应用/终端数据库模型
- 客户端插件（Windows/Linux）
- 硬件驱动层（加密卡、随机数发生器）
- 内置 CA 中心（若独立进程）

---

## 3. 总体架构设计

### 3.1 系统分层

```
┌─────────────────────────────────────────────────────────┐
│                    外部用户流量入口                       │
│          (HTTPS / TCP / UDP - 来自 DMZ 外网)              │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │    OpenResty 引擎层      │  ◀── 核心改造
        │   (Nginx + Lua)         │
        │                         │
        ├─────────────────────────┤
        │ Nginx 核心层            │
        ├─────────────────────────┤
        │ · SSL/TLS 终结          │
        │ · HTTP 反向代理         │
        │ · Stream TCP/UDP 代理   │
        │ · 连接复用、缓冲、压缩  │
        │ · 负载均衡 (upstream)   │
        ├─────────────────────────┤
        │ Lua 业务层              │
        ├─────────────────────────┤
        │ · 认证鉴权              │
        │ · RBAC 策略执行         │
        │ · SSO 表单代填/证书透传 │
        │ · 终端安全基线检查      │
        │ · 日志采集              │
        │ · 统一门户              │
        ├─────────────────────────┤
        │ 共享缓存层              │
        ├─────────────────────────┤
        │ · lua_shared_dict       │
        │   - 在线用户 Session    │
        │   - CRL 缓存            │
        │   - 策略规则缓存        │
        │   - 连接计数器          │
        └─────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │    后端服务层 (不变)     │
        │                         │
        ├─────────────────────────┤
        │ · 用户/角色/资源管理    │
        │ · 策略配置存储          │
        │ · 证书管理 (CA/OCSP)   │
        │ · 审计日志存储          │
        │ · 第三方认证代理        │
        │ · 高可用管理            │
        └─────────────────────────┘
```

### 3.2 关键架构决策

| 决策 | 选择 | 理由 | 风险 |
|------|------|------|------|
| D1：SSL 层处理 | OpenResty 原生 SSL | 性能最优，编译进 Nginx | 国密 OpenSSL 编译兼容性 |
| D2：国密支持 | Tongsuo 或国密 OpenSSL | 在 C 层支持 SM2/SM4，不走 Lua | Phase 0 需验证 |
| D3：认证逻辑 | 核心认证在 Lua，第三方评估 | 减少网络往返 | LDAP/RADIUS 库成熟度 |
| D4：会话管理 | shared_dict + 后端双写 | 高性能读取 + 持久化 + 热备同步 | 同步机制复杂度 |
| D5：配置管理 | 后端推送 → shared_dict 缓存 | 避免修改 Nginx 配置文件 | 热加载延迟 |
| D6：NC 隧道 | stream 模块 + Lua 控制面 | 协议兼容性最高 | 数据面性能评估 |
| D7：SSO 代填 | Lua body_filter 解析 HTML | 简单易维护 | 复杂 JS 表单支持范围有限 |
| D8：双机热备 | VRRP + shared_dict 同步 | 支持无缝切换 | 状态一致性保证 |

---

## 4. 模块架构详设

### 4.1 模块全景与职责

```
┌────────────────────────────────────────────────────────────┐
│                   TSG 7.6 核心模块体系                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │         OpenResty 引擎层（10个模块）                   │ │
│  │                                                      │ │
│  │  M1: SSL/TLS 终结模块 (ngx.ssl + 国密补丁)          │ │
│  │    └─ 职责：握手、证书验证、CRL/OCSP 检查            │ │
│  │                                                      │ │
│  │  M2: 反向代理模块 (proxy_pass + rewrite)             │ │
│  │    └─ 职责：B/S 应用转发、URL 重写、Cookie 改写     │ │
│  │                                                      │ │
│  │  M3: 认证鉴权模块 (access_by_lua)                   │ │
│  │    └─ 职责：证书/口令/双因子认证、第三方认证         │ │
│  │                                                      │ │
│  │  M4: RBAC 策略执行模块 (access_by_lua)              │ │
│  │    └─ 职责：角色匹配、权限判断、策略缓存             │ │
│  │                                                      │ │
│  │  M5: SSO 单点登录模块 (body_filter_by_lua)          │ │
│  │    └─ 职责：表单代填、证书透传、Cookie 跨域         │ │
│  │                                                      │ │
│  │  M6: NC 隧道模块 (stream 模块 + Lua)                │ │
│  │    └─ 职责：隧道建立、协议握手、数据转发             │ │
│  │                                                      │ │
│  │  M7: 统一门户模块 (content_by_lua)                  │ │
│  │    └─ 职责：门户页面渲染、应用列表展示               │ │
│  │                                                      │ │
│  │  M8: 终端安全检查模块 (access_by_lua)               │ │
│  │    └─ 职责：终端指纹验证、基线检查、绑定校验        │ │
│  │                                                      │ │
│  │  M9: 日志采集模块 (log_by_lua)                      │ │
│  │    └─ 职责：访问日志记录、SYSLOG 转发                │ │
│  │                                                      │ │
│  │  M10: 会话管理模块 (shared_dict + lua-resty)        │ │
│  │    └─ 职责：在线用户状态、超时注销、会话同步         │ │
│  │                                                      │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │      后端服务层（6 个模块，保持不变）                  │ │
│  │                                                      │ │
│  │  M11: 用户与资源管理 (CRUD / 批量操作)              │ │
│  │  M12: 策略管理 (配置存储 / 下发)                    │ │
│  │  M13: 证书管理 (CA 签发 / CRL 更新)                 │ │
│  │  M14: 审计日志 (存储 / 查询 / 导出)                 │ │
│  │  M15: 第三方认证代理 (RADIUS / LDAP / AD)           │ │
│  │  M16: 高可用管理 (双机热备 / 状态同步)              │ │
│  │                                                      │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │      管理面层（保持不变）                              │ │
│  │                                                      │ │
│  │  M17: Web UI (三权分立 / 配置管理 / 监控)           │ │
│  │                                                      │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 4.2 模块职责详表

#### 引擎层（OpenResty）

| 模块 | 功能 | 关键实现 | 输入/输出 | 依赖 |
|------|------|---------|----------|------|
| M1 | 国际 SSL 握手 + 国密 SSL 握手 + 证书验证 + CRL/OCSP 检查 | ngx.ssl + ssl_certificate_by_lua | 客户端证书 → 验证结果 | M13 |
| M2 | HTTP 路由 + URL 重写 + 后端转发 + Cookie 改写 + 压缩 | proxy_pass + rewrite_by_lua + gzip | HTTP 请求 → HTTP 响应 | - |
| M3 | 证书认证 + 口令认证 + 双因子 + LDAP/RADIUS | access_by_lua + ssl_certificate_by_lua | 凭据 → 用户身份 | M15、M13 |
| M4 | 角色提取 + 资源匹配 + 时间/IP/端口判断 + 动态规则匹配 | access_by_lua + shared_dict | 用户身份 + 请求 → 访问决策 | M12 |
| M5 | HTML 表单代填 + Header 注入 + Cookie 跨域处理 | body_filter_by_lua + header_filter_by_lua | 响应体 → 修改后的响应体 | - |
| M6 | 隧道建立 + 协议握手 + 策略校验 + 点对点/星型转发 | stream 模块 + preread_by_lua | TCP/UDP 连接 → 隧道数据 | M3, M4 |
| M7 | 门户 HTML 渲染 + 授权应用列表查询 + 跳转 | content_by_lua + 简易模板 | HTTP GET / → 门户页面 HTML | M4 |
| M8 | 终端指纹提取 + 绑定校验 + 安全基线检查 | access_by_lua + Lua 规则 | 终端信息 Header → 校验结果 | M11、后端 |
| M9 | 访问日志记录 + 字段映射 + 本地文件 + SYSLOG 转发 | log_by_lua | 请求/响应元数据 → 日志行 | M14 |
| M10 | 会话创建 + 有效期管理 + 超时注销 + KEY 拔出注销 + 热备同步 | lua-resty-session + shared_dict | 用户身份 → session_id + 会话数据 | 后端 DB |

#### 后端服务层（保持不变）

| 模块 | 功能 | 说明 |
|------|------|------|
| M11 | 用户/角色/应用/终端 CRUD 和批量导入导出 | 数据库 + REST API |
| M12 | 访问策略、安全策略的配置存储和版本管理 | 数据库 + 推送机制 |
| M13 | CA 签发、CRL 定时更新、OCSP 验证 | 与现有 CA 系统集成 |
| M14 | 日志存储、分类查询、导出、SYSLOG 聚合 | 日志库 + 查询接口 |
| M15 | RADIUS/LDAP/AD 认证协议对接和代理 | 协议转发 API |
| M16 | 双机热备配置管理、会话状态同步 | DB 主从 + 状态推送 |

---

## 5. 核心业务流程设计

### 5.1 用户登录与访问流程

```
用户请求 (HTTPS)
  │
  ├─→ [M1: SSL/TLS 终结]
  │   ├─ 建立 SSL 连接（协商算法、交换密钥）
  │   └─ 如果双向认证，验证客户端证书（调 M13 CRL/OCSP）
  │
  ├─→ [M3: 认证鉴权]
  │   ├─ 检查 Session Cookie（查 M10 会话）
  │   │   ├─ 若 Session 有效 → 跳转到步骤 4
  │   │   └─ 若 Session 无效或不存在 → 跳转到步骤 2
  │   │
  │   ├─ [第 2 步] 进行身份认证
  │   │   ├─ 若证书认证 → ssl_certificate_by_lua 验证（M13 协助）
  │   │   ├─ 若口令认证 → 转发至后端 API (M15 LDAP/RADIUS)
  │   │   └─ 若双因子 → 证书 + 口令组合验证
  │   │
  │   ├─ 认证成功 → 生成 session_id，存入 M10 (shared_dict)
  │   │   会话数据结构：
  │   │   {
  │   │     user_id = "xxx",
  │   │     roles = {"admin", "operator"},
  │   │     login_time = 1746600000,
  │   │     expire_time = 1746603600
  │   │   }
  │   │
  │   └─ 返回 Set-Cookie: session_id=xxx
  │
  ├─→ [M4: RBAC 策略执行]
  │   ├─ 提取用户角色（从 M10 会话中）
  │   ├─ 查询 M12 策略规则（缓存在 shared_dict）
  │   ├─ 根据「角色 + 应用资源 + 时间 + IP」判断访问权限
  │   │   ├─ 无权 → 返回 403 Forbidden
  │   │   └─ 有权 → 继续
  │   │
  │   └─ 可选：执行动态角色匹配（基于证书 CN/OU、源 IP 段等）
  │
  ├─→ [M8: 终端安全检查]
  │   ├─ 提取客户端上报的终端指纹（HTTP Header 或独立 API）
  │   ├─ 查询 M11 终端绑定表
  │   │   ├─ 若未绑定 → 拒绝或要求绑定
  │   │   └─ 若已绑定 → 检查安全基线
  │   ├─ 调用后端 API 校验终端安全基线（可选 COS 检查）
  │   │   ├─ 基线合格 → 继续
  │   │   └─ 基线不合格 → 返回 403
  │   │
  │   └─ 可选：指导客户端关闭非认证网卡（双网隔离实现）
  │
  ├─→ [第 4 步] 转发请求到后端应用
  │   ├─ [M2: 反向代理]
  │   │   ├─ 执行 URL 重写（如 /app/real/* → /real/*)
  │   │   ├─ 添加透传 Header（如 X-Forwarded-For）
  │   │   └─ proxy_pass 到后端应用 upstream
  │   │
  │   ├─ [M5: SSO 单点登录]（如果配置了代填）
  │   │   ├─ 捕获响应 body
  │   │   ├─ 若响应含 <form>，在提交前注入隐藏表单字段
  │   │   │   <input type="hidden" name="user" value="xxx">
  │   │   └─ 返回修改后的 HTML
  │   │
  │   └─ 返回响应给客户端
  │
  └─→ [M9: 日志采集]
      ├─ 在 log_by_lua 阶段，收集请求/响应元数据
      ├─ 日志格式：
      │   timestamp|user|src_ip|dest_ip|dest_port|method|url|status|bytes|duration|terminal_id|...
      │
      ├─ 写入本地日志文件（滚动）
      └─ 可选：通过 lua-resty-socket 转发到 SYSLOG 服务器
```

### 5.2 NC 模式隧道建立流程

```
客户端发起 TCP/UDP 连接 (端口 8443)
  │
  ├─→ [M1: SSL/TLS 握手]
  │   └─ 建立安全连接
  │
  ├─→ [M3: 认证鉴权]
  │   ├─ 提取证书或口令
  │   └─ 验证客户端身份 → 分配用户 Session
  │
  ├─→ [M4: RBAC 策略]
  │   ├─ 检查用户是否有权使用 NC 隧道
  │   ├─ 检查目标资源 (隧道端口范围)
  │   └─ 检查时间/IP 限制
  │
  ├─→ [M6: NC 隧道建立]
  │   ├─ 使用 stream 模块 + preread_by_lua
  │   ├─ 解析隧道协议头，获取目标地址/端口
  │   ├─ 建立到后端服务的 socket
  │   └─ 启动双向数据转发（对端网关 或 内部服务）
  │
  ├─→ 隧道数据转发 (完全透传)
  │   ├─ 发送数据 (客户端 → 网关 → 后端)
  │   └─ 接收数据 (后端 → 网关 → 客户端)
  │
  └─→ [M9: 日志采集]
      ├─ 记录隧道建立/关闭事件
      └─ 统计数据字节数
```

### 5.3 管理面配置下发流程

```
管理员修改策略 (Web UI)
  │
  ├─→ M17 (管理控制台) 提交配置修改
  │   └─ HTTPS 请求到后端管理 API
  │
  ├─→ M12 (策略管理) 持久化到数据库
  │
  ├─→ M16 (高可用管理) 推送变更到 OpenResty
  │   ├─ 推送到主节点 shared_dict （lua-resty + HTTP API）
  │   └─ 推送到备节点 shared_dict
  │
  └─→ OpenResty 热加载
      ├─ Lua 代码定期检查 shared_dict 中的版本号
      ├─ 若版本号更新，重新加载策略规则
      └─ 已有连接不中断，仅新连接使用新规则
      （或者执行 `nginx -s reload` 平滑重启）
```

---

## 6. 关键技术方案

### 6.1 SSL/TLS 与国密支持

**设计要点**：
1. **国密 OpenSSL 编译**：使用 Tongsuo（阿里开源国密分支）或国密版 OpenSSL
2. **双证书支持**：配置 RSA 和 SM2 双证书，根据客户端协商选择
3. **自定义证书验证**：在 ssl_certificate_by_lua 中实现 CRL/OCSP 检查
4. **算法协商**：支持 ECDHE-SM2-SM4-CBC-SM3、ECDHE-RSA-AES128-GCM-SHA256 等混合套件

**配置示例**（伪代码）：
```nginx
server {
    listen 443 ssl;
    
    # RSA 证书
    ssl_certificate /certs/rsa_cert.pem;
    ssl_certificate_key /certs/rsa_key.pem;
    
    # SM2 国密证书
    ssl_certificate /certs/sm2_cert.pem;
    ssl_certificate_key /certs/sm2_key.pem;
    
    # 混合加密套件
    ssl_ciphers "ECDHE-SM2-SM4-CBC-SM3:ECDHE-RSA-AES128-GCM-SHA256:...";
    
    # 自定义证书验证
    ssl_certificate_by_lua_file /lua/cert_verify.lua;
}
```

**Lua 证书验证逻辑（伪代码）**：
```lua
-- cert_verify.lua
local cert_chain = ngx.var.ssl_client_cert
if cert_chain then
    -- 1. 验证证书链
    -- 2. 检查 CRL（从 M13 后端拉取）
    -- 3. 检查 OCSP（可选）
    -- 4. 验证有效期
    -- 5. 验证用途（如 TLS Client Auth）
    if not verify_success then
        return ngx.ERROR  -- 拒绝连接
    end
end
```

### 6.2 认证与会话管理

**多认证源支持**：
1. **证书认证**：提取 CN、OU、SN 等字段映射到用户
2. **口令认证**：本地数据库或 LDAP/AD/RADIUS 校验
3. **双因子**：证书 + 口令、口令 + 短信等组合

**会话数据结构**（Lua table）：
```lua
local session = {
    user_id = "user123",
    name = "张三",
    roles = {"admin", "operator"},
    login_time = 1746600000,
    expire_time = 1746603600,  -- 当前时间 + 30 分钟
    terminal_id = "mac_xxxxx",
    auth_method = "cert",  -- 或 "password", "ldap" 等
    ip = "192.168.1.100"
}
```

**会话存储与同步**：
- **主机**：会话存在 shared_dict，同时定期异步写入后端数据库
- **备机**：通过定时 HTTP API 从主机拉取会话快照，或使用 Redis 外部存储
- **过期策略**：shared_dict 设置 TTL，加上 Lua 手动检查

### 6.3 RBAC 策略执行

**规则缓存结构**（在 shared_dict 中）：
```lua
role_policies = {
    ["admin"] = {
        apps = {"/app1/*", "/app2/api"},
        ips = {"192.168.1.0/24"},
        ports = {443, 8443},
        time_ranges = {"09:00-17:00"}
    },
    ["operator"] = {
        apps = {"/app1/view"},
        ips = {"*"},
        ports = {443},
        time_ranges = {"*"}
    }
}
```

**Lua 策略匹配伪代码**：
```lua
-- 在 access_by_lua 中执行
local user_roles = ngx.ctx.session.roles
local request_uri = ngx.var.uri
local remote_addr = ngx.var.remote_addr
local current_hour = ngx.localtime().hour

for role in pairs(user_roles) do
    local policy = role_policies[role]
    if matches_app(request_uri, policy.apps) and
       matches_ip(remote_addr, policy.ips) and
       matches_time(current_hour, policy.time_ranges) then
        return ngx.OK  -- 允许
    end
end

ngx.exit(ngx.HTTP_FORBIDDEN)  -- 拒绝
```

### 6.4 SSO 表单代填

**实现原理**（Lua body_filter）：
1. 捕获响应 body（使用 ngx.arg[1]）
2. 使用正则或 HTML 解析库找到 `<form>` 标签
3. 在 `</form>` 前注入隐藏表单字段
4. 修改 Content-Length header

**示例代码**（伪代码）：
```lua
-- sso_form.lua
local body = ngx.arg[1]
local need_inject = need_sso_injection(ngx.var.uri)

if need_inject and body then
    local session = ngx.ctx.session
    local hidden_field = string.format(
        '<input type="hidden" name="user" value="%s">',
        session.name
    )
    body = body:gsub("</form>", hidden_field .. "</form>")
    ngx.arg[1] = body
    ngx.var.content_length = string.len(body)
end
```

### 6.5 NC 隧道协议

**隧道建立流程**：
1. 客户端通过 SSL/TLS 建立到网关的控制连接
2. 发送隧道协议头 (包含目标地址、端口、隧道 ID)
3. 网关在 preread_by_lua 中解析协议头
4. 根据策略（M4）判断是否允许
5. 建立到后端的数据连接
6. 启用双向数据转发

**协议头格式**（保持与原 Java 版本兼容）：
```
[隧道类型(1字节)] [目标地址类型(1字节)] [目标地址长度(2字节)] [目标地址] [端口(2字节)]

示例：
0x01 0x01 0x0B "192.168.1.100" 0x0D05 (端口 3333)
```

### 6.6 高可用与双机热备

**部署拓扑**：
```
                   ┌─────────────────┐
                   │   VRRP 虚拟 IP   │
                   │  (对外服务地址)   │
                   └────────┬──────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │ 优先级高          │ 优先级低           │
        ▼                   ▼                   │
   ┌──────────┐        ┌──────────┐           │
   │ 主节点    │        │ 备节点    │           │
   │ (MASTER) │        │(BACKUP)  │           │
   │          │        │          │           │
   │Nginx+Lua │        │Nginx+Lua │           │
   │          │ rsync  │          │           │
   │ shared_ ◄├───────►│ shared_  │           │
   │  dict    │  配置  │  dict    │           │
   └──────┬───┘  +会话 └───┬──────┘           │
          │      同步      │                   │
          └────────┬───────┘                   │
                   │                           │
            ┌──────▼──────┐                    │
            │ MySQL Master│                    │
            │   (主库)     │                    │
            └──────┬──────┘                    │
                   │                           │
            ┌──────▼──────┐                    │
            │MySQL Slave  │                    │
            │  (从库)      │                    │
            └─────────────┘                    │
                                               │
                   灰度切换：通过 DNS / LB 调整流量比例
```

**会话同步机制**：
- **Option A**（推荐）：主节点异步写 DB，备节点定时拉取
- **Option B**（可选）：使用 Redis 作为外部 session 存储
- **切换时间**：≤ 3 秒（VRRP 检测间隔）

---

## 7. 性能指标与非功能性设计

### 7.1 性能目标

| 指标 | 原 Java 版本 | OpenResty 目标值 | 基准硬件 |
|------|-------------|------------------|---------|
| 国密新建连接数（次/秒） | 4,500 | ≥ 4,500 | SG-5600/6000 |
| 国密 SSL 事务处理速率（次/秒） | 8,000 | ≥ 8,000 | - |
| 国密最大并发连接数 | 50,000 | ≥ 50,000 | - |
| 国密加密带宽（Mbps） | 1,200 | ≥ 1,200 | - |
| HTTP 请求处理延迟（P99，ms） | 50 | ≤ 60（不高于 120%） | - |
| SSL 握手延迟（P99，ms） | 70 | ≤ 70（不劣化） | - |
| 空闲内存占用 | 1.5-2.0 GB | ≤ 1.4 GB（70%） | - |
| 支持接入用户数 | 15,000 | ≥ 15,000 | - |

### 7.2 资源占用目标

| 项目 | 要求 | 说明 |
|------|------|------|
| 启动内存 | ≤ Java 版本 70% | 重点优化：关闭不必要的 Lua 库、缩小 shared_dict 大小 |
| 运行内存 | ≤ Java 版本 75% | 考虑缓存膨胀、活跃连接数 |
| CPU 效率 | 同等压力下，CPU 使用率 ≤ Java | 充分利用多 worker 并行 |
| 磁盘 I/O | 与原版本持平 | 日志吞吐不增加 |

### 7.3 可靠性设计

- **进程隔离**：Nginx master + 多个 worker，单 worker 崩溃不影响全局
- **Lua 沙箱**：LuaJIT 沙箱隔离，防止恶意代码；私钥仅在 worker 内存中，不进入 shared_dict
- **配置热重载**：`nginx -s reload` 平滑替换 worker，已连接不中断
- **故障转移**：VRRP 检测主节点失败，自动切换到备节点（≤ 3 秒）

### 7.4 安全性设计

| 防护项 | 实现方式 |
|--------|---------|
| 限流防暴力 | `limit_req` 模块限制请求速率 |
| IP 黑白名单 | access_by_lua 查询 shared_dict 或数据库 |
| 防 SQL 注入 | Lua 层对动态 SQL 参数转义（若涉及） |
| 国密合规 | 所有国密算法通过 Tongsuo FIPS 或国密认证 |
| 日志防篡改 | 日志写入后端存储，OpenResty 层不持久化 |

---

## 8. 技术栈与开源库

| 组件 | 版本 | 选型原因 | 备注 |
|------|------|---------|------|
| OpenResty | 1.21.4.1+ | 集成 Nginx 1.21 + LuaJIT 2.1，成熟稳定 | - |
| Nginx | 1.21+ | 原生 HTTP/stream、高性能 I/O | - |
| LuaJIT | 2.1 | 性能优于 Lua 5.1，广泛应用 | - |
| 国密 OpenSSL | Tongsuo 8.3.0+ | 阿里开源，支持 SM2/SM3/SM4 | 编译进 Nginx |
| lua-resty-session | 最新 | 分布式会话管理库 | 可选 |
| lua-resty-http | 最新 | HTTP 客户端，用于调用后端 API | 必需 |
| lua-resty-socket | 最新 | UDP/TCP socket，用于 SYSLOG 转发 | 必需 |
| lua-resty-lrucache | 最新 | 本地 LRU 缓存，加速规则匹配 | 可选 |

---

## 9. 迁移与兼容性策略

### 9.1 配置迁移

**工具**：Python 脚本 + Jinja2 模板
- 输入：原 Java 网关配置 (XML/JSON)
- 处理：解析应用、端口、SSL 证书、用户策略
- 输出：`nginx.conf` + `conf.d/*.conf` + `lua/*.lua` + 初始化脚本

**迁移清单**：
```
原配置项          → OpenResty 映射
web.xml           → nginx.conf (location 块)
applicationXML    → lua/app_routing.lua
securityPolicy    → lua/rbac.lua
ssoConfig         → lua/sso_form.lua
auditConfig       → lua/access_log.lua
```

### 9.2 客户端兼容性

**验证范围**：
- Windows ActiveX 控制件（原有版本）
- Linux 客户端（C/S 应用）
- 浏览器插件

**验证要点**：
- SSL 握手：国际 + 国密两种协议
- 双向认证：证书验证逻辑一致
- NC 隧道：隧道协议完全兼容
- SSO：表单注入不影响 JS 运行

**预期结果**：无需升级客户端

### 9.3 灰度发布计划

```
阶段 1: 内网小流量测试（1% 用户）
  └─ 持续 1 周，监控性能和错误率
  
阶段 2: 增加到 10% 用户
  └─ 持续 2 周，确保稳定性
  
阶段 3: 增加到 50% 用户
  └─ 持续 1 周，评估是否全量切换
  
阶段 4: 全量切换（100%）
  └─ 主网关切换，保留原 Java 版本作为备份
  
阶段 5: 上线后 1 个月
  └─ 监控和优化，确认无回退需求后下架 Java 版本
```

---

## 10. 实施阶段规划

### 10.1 阶段划分

| 阶段 | 内容 | 产出 | 预估周期 | 优先级 |
|------|------|------|----------|--------|
| **Phase 0** | 国密 PoC 验证 + 技术可行性评估 | PoC demo + 可行性报告 | 1 周 | 🔴 P0 |
| **Phase 1** | 核心流量迁移（SSL + 反向代理 + 认证 + RBAC） | 功能完整性验证 | 4-6 周 | 🔴 P0 |
| **Phase 2** | 高级功能迁移（SSO + NC + 双机热备） | 完整功能集 | 3-4 周 | 🟡 P1 |
| **Phase 3** | 监控审计适配（日志 + SNMP + SysLog） | 审计日志完整 | 2 周 | 🟡 P1 |
| **Phase 4** | 全量功能 + 性能测试 | 性能基准报告 | 2-3 周 | 🟡 P1 |
| **Phase 5** | 兼容性验证 + 灰度试运行 | 验收通过 | 1-2 周 | 🟢 P2 |

**总预计周期**：6-7 个月

### 10.2 风险与应对

| 风险 | 等级 | 影响 | 应对措施 |
|------|------|------|----------|
| 国密 OpenSSL 与 Nginx 编译兼容性 | 🔴 高 | Phase 0 阻断 | PoC 验证 + 备选方案（分离进程） |
| Lua 第三方库（LDAP/RADIUS）成熟度不足 | 🟡 中 | 影响第三方认证 | 优先使用后端 API 转发方案 |
| SSO 表单代填对复杂 JS 表单支持 | 🟡 中 | 功能缺陷 | Phase 1 支持简单表单，Phase 2 评估扩展 |
| 双机热备会话同步一致性 | 🟡 中 | 主备切换后会话丢失 | 采用后端 DB 持久化 + 定期拉取 |
| 性能未达预期 | 🟢 低 | 回退需求 | 保留 Java 版本 1 个月，持续优化 |

---

## 11. 验收标准

### 11.1 功能验收（覆盖率 100%）

执行以下测试用例集合：
- **白皮书功能清单**：所有 ~150+ 项功能
- **关键业务流程**：登录→代理→SSO→日志（完整链路）
- **边界情况**：会话超时、证书过期、策略变更热加载

**验收标准**：所有测试用例与原 Java 版本结果一致

### 11.2 性能验收

**测试工具**：LoadRunner + Scapy + 国密 SSL 脚本

**测试场景**：
1. SSL 握手测试：逐步增加并发，测试新建连接数、握手延迟
2. HTTP 代理测试：混合 HTTPS 短连接 + 长连接，测试吞吐和延迟
3. 混合场景测试：SSL + HTTP 代理 + 部分 NC 隧道，逐步加压至目标并发 120%

**验收标准**：各项指标 ≥ 基线值（见 7.1 表）

### 11.3 稳定性验收

- **7×24 小时压力测试**：逐步加压至目标并发 120%，无 crash、内存泄漏、句柄泄漏
- **1000 次热重载测试**：每次执行 `nginx -s reload`，验证已连接不中断
- **备机切换测试**：模拟主机故障，验证切换时间 ≤ 3 秒，会话保留

### 11.4 迁移验收

使用真实客户配置：
- 配置转换工具处理 ≥ 3 个不同配置规模
- 验证应用、用户、策略、证书完整恢复
- 访问行为与旧网关完全一致

### 11.5 兼容性验收

- **客户端测试**：Windows ActiveX、Linux C/S、浏览器插件均可正常使用
- **证书验证**：双向认证、证书链验证、CRL/OCSP 检查一致
- **NC 隧道**：隧道协议兼容，无需客户端升级

---

## 12. 其他关键设计点

### 12.1 日志审计完整性

**日志格式** (完全兼容原版)：
```
timestamp|user|src_ip|dest_ip|dest_port|method|uri|status|bytes_sent|duration_ms|terminal_id|auth_type|result
2026-05-07 14:30:45|user123|192.168.1.100|10.0.0.20|443|GET|/app1/api|200|1024|25|mac_xxxxx|cert|success
```

**日志路由**：
1. 本地文件 `/var/log/tsg/access.log`（按天滚动）
2. 可选：SYSLOG 服务器（UDP 转发）
3. 后端审计库（异步写入）

### 12.2 在线用户与连接管理

**shared_dict 统计项**：
- `online_users_count`：当前登录用户数
- `active_connections`：活跃连接数
- `ssl_handshakes_total`：累计 SSL 握手次数
- `ssl_handshakes_rate`：当前握手速率（次/秒）

**查询接口**：`GET /_status` 返回 JSON 格式统计

### 12.3 监控与告警集成

**暴露的指标** (Prometheus 格式)：
- `tsg_active_connections`
- `tsg_ssl_handshake_duration_ms`
- `tsg_http_request_duration_ms`
- `tsg_memory_used_bytes`
- `tsg_authentication_failures_total`

**告警规则示例**：
- 连接数 > 容量 80%：黄色告警
- 内存占用 > 80%：黄色告警
- 握手延迟 P99 > 100ms：黄色告警
- worker 进程 crash：红色告警

---

## 13. 开发与测试建议

### 13.1 开发建议

1. **模块化编码**：
   - 每个 Lua 模块（M1~M10）独立成文件（如 `lua/auth.lua`、`lua/rbac.lua`）
   - 提供清晰的函数接口，便于单元测试

2. **配置管理**：
   - 使用 Lua 全局 table 管理配置，支持热加载
   - 敏感信息（密钥、密码）不放 shared_dict，仅放环境变量或后端 API

3. **错误处理**：
   - 所有外部 API 调用加超时控制（防止后端超时拖累网关）
   - 认证失败等关键路径加详细日志

### 13.2 测试建议

1. **单元测试**：
   - 对 Lua 模块使用 `busted` 框架编写单元测试
   - 测试覆盖率 ≥ 80%

2. **集成测试**：
   - 搭建完整测试环境（后端应用 + OpenResty 网关）
   - 使用 Postman / curl 脚本覆盖各业务流程

3. **性能基准**：
   - 建立原 Java 版本的性能基准（握手速率、吞吐、延迟）
   - 使用相同工具和场景对 OpenResty 版本进行压测

4. **兼容性测试**：
   - 真实客户端环境测试（Windows/Linux/浏览器）
   - 证书、隧道等关键路径验证

---

## 14. 一句话结论与建议

### 结论

> **TSG 7.6 架构升级的核心是"换引擎不换功能"**：用 OpenResty (Nginx + Lua) 替换 Java 对外服务引擎，保证功能完全等价、性能显著提升（内存占用降低 30%、并发处理能力提升 30%~50%）、客户端无感知。设计已明确 10 个引擎层模块、6 个后端模块的职责边界，关键风险点（国密 SSL、NC 隧道兼容性）已识别，建议按 Phase 0~5 逐步实施。

### 建议

1. **优先启动 Phase 0**（国密 PoC）：这是整个项目的先决条件，如果不能解决国密编译问题，后续工作无法进行。

2. **明确 NC 隧道数据面方案**：决定是使用 OpenResty stream 模块，还是保留外部进程处理，影响 Phase 1 的分工。

3. **建立性能基准**：在开发前，使用原 Java 版本建立详细的性能基准（握手速率、延迟分布、内存占用等），便于后续对标。

4. **预留回退方案**：灰度发布期间保留原 Java 版本，确保问题发生时可快速切回。

5. **定期同步进度**：由于改造复杂度较高，建议每周与产品、测试团队进行进度同步和风险评估。

---

## 附录 A：功能完整性清单

以下功能必须 100% 保留，底层实现更改但行为完全一致：

- ✅ 数据传输加密（国密 SSL + 国际 SSL）
- ✅ 双网隔离（客户端关闭非认证网卡）
- ✅ 数字证书认证（含 CRL/OCSP/链式验证）
- ✅ 多因子认证（证书 + 口令、口令 + 短信等）
- ✅ 内置 CA 中心（证书生成/签发/吊销）
- ✅ 最小访问权限原则（RBAC）
- ✅ 权限动态匹配（黑白名单、基于属性的规则）
- ✅ 终端安全检查（指纹提取、基线验证、绑定）
- ✅ 零痕迹访问（退出时清除客户端缓存）
- ✅ 应用/角色/用户/终端管理（CRUD + 批量操作）
- ✅ HTTP 压缩、高并发支持
- ✅ 双机热备、第三方负载均衡
- ✅ 单点登录（表单代填 + 证书透传）
- ✅ 统一门户（授权应用列表、快速导航）
- ✅ 三权分立（操作员/审计员/管理员）
- ✅ 审计日志（本地日志 + SYSLOG 转发 + 数据库存储）
- ✅ 实时监控（CPU/内存/连接数/握手速率等）
- ✅ 第三方认证（RADIUS/LDAP/AD/Kerberos）
- ✅ NC 模式（点对点 / 星型隧道拓扑）
- ✅ 动态端口应用（FTP/SIP 等控制流动态协议）

---

## 附录 B：关键术语与缩写

| 术语/缩写 | 解释 |
|----------|------|
| OpenResty | Nginx + LuaJIT 的集成发行版，用于快速开发高性能网络服务 |
| Tongsuo | 阿里开源的国密 OpenSSL 分支，支持 SM2/SM3/SM4 等国密算法 |
| shared_dict | OpenResty 中的共享内存缓存，跨 worker 进程可见 |
| access_by_lua | Nginx 的 access 阶段 Lua 钩子，用于权限检查 |
| body_filter_by_lua | Nginx 的 body 过滤阶段 Lua 钩子，用于响应体修改 |
| stream | Nginx 的流处理模块，支持 TCP/UDP 代理 |
| proxy_pass | Nginx 反向代理指令，转发请求到上游服务器 |
| CRL | Certificate Revocation List，证书吊销列表 |
| OCSP | Online Certificate Status Protocol，在线证书状态检查 |
| RBAC | Role-Based Access Control，基于角色的访问控制 |
| SSO | Single Sign-On，单点登录 |
| VRRP | Virtual Router Redundancy Protocol，虚拟路由冗余协议，用于双机热备 |
| LRU Cache | Least Recently Used 缓存，淘汰策略为最近最少使用 |
| Worker | Nginx 工作进程，处理实际请求 |

---

## 附录 C：相关文档引用

1. 《中宇万通安全网关-技术白皮书.pdf》
2. 《TSG 7.6 架构迁移确认书 v1.1》
3. OpenResty 官方文档：http://openresty.org/cn/
4. Nginx 官方文档：http://nginx.org/
5. Tongsuo 官方仓库：https://github.com/Tongsuo-Project/Tongsuo

---

**文档批准**：

| 角色 | 签名 | 日期 |
|------|------|------|
| 技术负责人 | __________ | __________ |
| 产品经理 | __________ | __________ |
| 测试经理 | __________ | __________ |

---

**版本历史**：

| 版本 | 日期 | 变更 | 作者 |
|------|------|------|------|
| v1.0 | 2026-05-07 | 初稿（深度详细版） | 团队 A |
| v1.1 | 2026-05-07 | 精简概览版 | 团队 B |
| v1.2 | 2026-05-07 | 结构化版本 | 团队 C |
| v2.0 | 2026-05-07 | **合并版**（本文档） | 架构设计团队 |

