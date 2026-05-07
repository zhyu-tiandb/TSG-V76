markdown
# 概要设计：TrustMore 安全网关架构升级（Java → OpenResty）

| 文档版本 | 1.0 | 创建日期 | 2026-05-07 |
|---------|-----|----------|-------------|
| 产品名称 | TrustMore 安全网关（TSG 7.6） | 设计类型 | 架构重构 |
| 设计负责人 | 田东波 | 评审级别 | 内部评审 |

---

## 1. 设计目标

- **引擎替换**：将对外服务引擎从 Java 替换为 OpenResty（Nginx + LuaJIT）
- **功能等价**：所有现有功能行为不变，用户/管理员无感知
- **性能提升**：并发连接数、SSL 吞吐、握手延迟均优于 Java 版本
- **国密完整**：支持 SM2/SM3/SM4 及国密 SSL 协议
- **平滑迁移**：支持现有配置导入，客户端无需升级

---

## 2. 总体架构

### 2.1 部署视图（不变）

```text
[外网用户] → [防火墙] → [TrustMore 安全网关（OpenResty）] → [内网应用]
                              │
                              ├─ 反向代理模式（B/S、C/S）
                              ├─ NC 模式（TCP/UDP 隧道）
                              └─ 管理端（后端 Java/Go 不变）
网关仍位于 DMZ 区，支持主路/旁路部署

管理端（三权分立、配置管理、日志查看）保持不变，仅调用 OpenResty 提供的内部 API 或直接读取数据库

2.2 逻辑架构（OpenResty 内部）

┌─────────────────────────────────────────────────────────────────────┐
│                         OpenResty (Nginx + Lua)                      │
├───────────────────┬───────────────────┬───────────────────┬─────────┤
│     SSL/TLS       │    HTTP/HTTPS     │      Stream        │   Lua   │
│     终结层        │      代理层       │      代理层        │  扩展层 │
│  (ngx_ssl/        │  (proxy_pass,     │  (stream           │ (access,│
│  国密补丁)        │   rewrite)        │   proxy)           │  log,..)│
├───────────────────┴───────────────────┴───────────────────┴─────────┤
│                        共享缓存 (lua_shared_dict)                     │
│                   会话状态 / CRL 缓存 / 动态策略缓存                   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                          ┌─────────────────────┐
                          │     后端服务（不变）   │
                          │  - 数据库（用户/配置） │
                          │  - 管理端 API         │
                          │  - 第三方认证（LDAP）  │
                          └─────────────────────┘
SSL/TLS 终结层：处理国际 TLS 和国密 SSL，证书验证、算法协商

HTTP/HTTPS 代理层：反向代理、URL 重写、压缩、统一门户

Stream 代理层：TCP/UDP 隧道（NC 模式）、C/S 应用代理

Lua 扩展层：认证、RBAC、动态角色、SSO 表单代填、日志记录、终端安全检查

共享缓存：存储在线用户 Session、CRL 数据、动态策略规则，跨 worker 进程共享

2.3 模块对应关系（Java → OpenResty）
Java 模块	OpenResty 对应	实现方式
SSLContext / 国密 SDK	OpenSSL + 国密补丁	编译进 Nginx
HTTP 代理处理器	proxy_pass + rewrite_by_lua	原生 + Lua
TCP/UDP 代理	stream 模块 + preread_by_lua	原生 + Lua
认证逻辑	access_by_lua_file	Lua 脚本
RBAC 授权	access_by_lua_file	Lua + shared_dict 缓存
动态角色匹配	access_by_lua_file	Lua 规则匹配
终端安全基线检查	access_by_lua_file + 后端 API	Lua 调用后端
SSO 表单代填	body_filter_by_lua_file	Lua 解析 HTML
日志审计	log_by_lua_file	Lua 写入文件 + SYSLOG
Session 管理	lua_resty_session + shared_dict	Lua 库
统一门户	content_by_lua_file	Lua 模板渲染
3. 核心模块设计
3.1 SSL/TLS 与国密支持
设计要点：

采用 Tongsuo（阿里开源的国密 OpenSSL 分支）或 国密版 OpenSSL 编译 Nginx

配置支持双证书（RSA + SM2），根据客户端协商选择加密套件

在 ssl_certificate_by_lua 阶段调用 Lua 脚本进行自定义证书验证（CRL、OCSP、证书链）

关键配置示例：

nginx
server {
    listen 443 ssl;
    ssl_certificate     /certs/sm2_cert.pem;
    ssl_certificate_key /certs/sm2_key.pem;
    ssl_certificate     /certs/rsa_cert.pem;
    ssl_certificate_key /certs/rsa_key.pem;
    
    ssl_ciphers "ECDHE-SM2-SM4-CBC-SM3:ECDHE-RSA-AES128-GCM-SHA256:...";
    ssl_certificate_by_lua_file /lua/cert_verify.lua;
}
3.2 用户认证与 Session 管理
设计要点：

统一在 access_by_lua 阶段处理认证

Session 存储在 lua_shared_dict 中，key 为 session_id，value 为用户信息、角色、过期时间

支持多认证源：本地口令、数字证书、RADIUS/LDAP/AD（Lua 直连或转发至后端 API）

支持双因子：证书 + 口令等组合验证

Session 数据结构：

lua
{
    user_id = "xxx",
    name = "张三",
    roles = {"admin", "operator"},
    login_time = 1746600000,
    expire_time = 1746603600,
    terminal_id = "mac_xxx",
    auth_method = "cert"
}
3.3 RBAC 与动态角色匹配
设计要点：

启动时从数据库加载角色-应用映射表到 shared_dict（定期刷新）

access_by_lua 中根据用户角色 + 请求 URL 判断是否允许访问

动态角色：基于用户属性（证书 CN、OU、IP 段）匹配白名单/黑名单规则，运行时计算角色

规则缓存结构：

lua
-- 白名单规则示例
role_rules[role_name] = {
    apps = {"/app1/*", "/app2/api"},
    ips = {"192.168.1.0/24"},
    time_ranges = {"09:00-17:00"}
}
3.4 终端安全检查与绑定
设计要点：

客户端在登录时上报终端指纹、基线信息（通过 HTTP Header 或独立 API）

OpenResty 调用后端验证接口（或直接查数据库）判断终端是否绑定、基线是否合格

不符合条件则返回 403，并携带错误码，客户端展示提示信息

双网隔离：认证通过后，OpenResty 在响应中下发指令，客户端关闭非认证网卡路由（客户端行为）

3.5 反向代理模式（B/S 应用）
设计要点：

使用 proxy_pass 直接转发

URL 重写：rewrite_by_lua 中修改 URI

Cookie 路径改写：header_filter_by_lua 中修改 Set-Cookie 字段

应用地址隐藏：统一门户中通过相对路径访问，OpenResty 代理到真实后端

示例：

nginx
location ~ ^/app/(.*)$ {
    rewrite_by_lua '
        local new_uri = "/real/" .. ngx.var[1]
        ngx.var.uri = new_uri
    ';
    proxy_pass http://backend_app;
}
3.6 NC 模式（TCP/UDP 隧道）
设计要点：

使用 stream 模块监听 SSL/TCP 端口

客户端先建立控制连接，进行认证和隧道协商

认证通过后，preread_by_lua 解析协议头，决定后端目标地址

数据面使用 proxy_pass 直接转发（或 SOCKS5 方式）

支持网状拓扑（点对点隧道）和星型拓扑（分支→总部）

配置概览：

nginx
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
3.7 SSO 表单代填
设计要点：

在 body_filter_by_lua 中修改响应 body

使用 Lua 的 gsub 或正则库在 </form> 前注入 <input type="hidden" name="user" value="xxx">

仅对配置中需要代填的 URL 进行处理

代填信息从当前 Session 中获取（用户明文口令需加密存储，或使用证书透传替代）

限制：复杂 JavaScript 生成的表单可能需要更高级的解析（可先支持简单 HTML Form，复杂场景推荐证书透传）。

3.8 统一门户
设计要点：

用户访问根路径 / 时，由 content_by_lua_file 动态生成门户页面

从 Session 获取用户角色，查询数据库（或缓存）得到授权应用列表，渲染 HTML

支持管理员定制 Logo、页面标题（通过后端配置）

支持认证后自动跳转到指定应用（302 重定向）

3.9 日志审计
设计要点：

在 log_by_lua 阶段记录访问日志

日志格式完全兼容原 Java 版本（字段顺序、分隔符）

同时写入本地文件（滚动）和通过 lua-resty-socket 发送到 SYSLOG 服务器

管理操作日志仍由后端记录，不经过 OpenResty

日志字段示例：

text
timestamp|user|src_ip|dest_ip|dest_port|method|url|status|bytes|duration|terminal_id|...
3.10 双机热备
设计要点：

主备两台网关共享一个虚拟 IP（VRRP）

配置使用 rsync/unison 同步（证书、配置文件、Lua 脚本）

会话状态同步：备机通过 API 定期从主机拉取 shared_dict 快照；或使用 Redis 作为外部 Session 存储（可选）

备机处于 standby 状态，不处理业务流量，但同步配置和状态

4. 关键技术选型
组件	选型	版本	备注
OpenResty	openresty	1.21.4.1+	包含 Nginx 1.21+ 和 LuaJIT
国密 SSL	Tongsuo	8.3.0+	替换 OpenResty 默认 OpenSSL
Lua 库	lua-resty-*	官方模块	session、lock、socket、redis
第三方认证	lua-resty-ldap / lua-resty-http	-	可选，也可走后端 API
日志发送	lua-resty-socket	-	发送 SysLog UDP
配置转换工具	Python 脚本	-	解析 Java XML/JSON，生成 Nginx conf + Lua
5. 数据流设计（关键流程）
5.1 用户登录流程
text
1. 客户端 → OpenResty：POST /login (证书/口令)
2. ssl_certificate_by_lua（若证书认证）：验证证书链、CRL/OCSP
3. access_by_lua：
   a. 提取凭据，调用认证模块（本地/第三方）
   b. 校验终端指纹（查询绑定表）
   c. 校验终端安全基线（调用后端或本地策略）
   d. 分配角色（静态或动态）
   e. 生成 session_id，存入 shared_dict
4. 返回 Set-Cookie + 门户页面重定向
5.2 访问受保护应用流程
text
1. 客户端携带 session cookie 请求 /app/xxx
2. access_by_lua：
   a. 验证 session 有效性（shared_dict 中存在且未超时）
   b. 根据用户角色 + 请求 URL 执行 RBAC 检查
   c. 可选：检查时间段、源 IP
3. 若通过：proxy_pass 到后端应用
4. 若需要 SSO 代填：body_filter_by_lua 修改响应
5. log_by_lua：记录访问日志
6. 配置与迁移设计
6.1 配置文件转换
原 Java 配置格式：XML（或 JSON）

转换工具读取原配置文件，生成：

nginx.conf：全局配置、监听端口、SSL 设置

conf.d/*.conf：各应用代理配置

lua/*.lua：认证、授权、SSO 等动态逻辑

shared_dict 初始化数据

6.2 迁移步骤
备份原 Java 网关配置

运行转换工具，输出 OpenResty 配置目录

复制证书文件到指定目录

启动 OpenResty，验证功能

灰度切换：通过 DNS/负载均衡将部分流量导入新网关

7. 非功能性设计
7.1 性能设计
多 worker 进程：利用多核 CPU，每个 worker 独立事件循环

连接池：proxy_pass 自带连接池，复用后端连接

缓冲调优：调整 proxy_buffer_size、proxy_buffers 适应大文件传输

SSL 会话缓存：ssl_session_cache shared:SSL:10m; 减少握手开销

7.2 可靠性设计
配置热重载：nginx -s reload 不重启 master，平滑替换 worker

健康检查：health_check 指令（商业版）或用 Lua 实现主动探测

熔断：可扩展实现后端失败重试、降级策略

7.3 安全性设计
限制请求速率：limit_req 防暴力破解

IP 黑白名单：access_by_lua 中查询数据库或配置

防 SQL 注入：对用户输入在 Lua 层转义（如果涉及动态 SQL）

国密合规：所有国密算法均通过 Tongsuo 的 FIPS 或国密认证

7.4 可维护性设计
日志分级：error.log、access.log 按天滚动

调试接口：开放 /_status 返回 shared_dict 统计和 worker 状态

提供 CLI：tsg-cli 工具用于配置检查、缓存清理、手动同步

8. 有待明确的问题（Open Items）
#	问题	负责人	截止日期
1	Tongsuo 与 OpenResty 的编译兼容性验证（Phase 0）	研发	2026-05-14
2	NC 模式隧道协议细节：是否需要保留原 Java 隧道协议完全兼容	研发	2026-05-14
3	SSO 表单代填对复杂 JS 表单的支持范围	产品+研发	2026-05-21
4	双机热备会话同步采用 pull 还是外部 Redis	研发	2026-05-21
5	现有客户最大配置规模（应用数、用户数）测试	测试	2026-05-28
9. 附录：模块迁移实现映射表（详细）
Java 类/功能	实现位置（OpenResty）	Lua 文件示例
SSLServerSocket	nginx.conf ssl 配置	-
X509Certificate 验证	ssl_certificate_by_lua	cert_verify.lua
UsernamePasswordAuthenticator	access_by_lua	auth_local.lua
LDAPAuthenticator	access_by_lua	auth_ldap.lua
SessionManager	shared_dict + lua-resty-session	session.lua
RBACFilter	access_by_lua	rbac.lua
TerminalBindFilter	access_by_lua	terminal_check.lua
ReverseProxyServlet	proxy_pass	-
URLRewriter	rewrite_by_lua	rewrite.lua
SSOFormFiller	body_filter_by_lua	sso_form.lua
AccessLogFilter	log_by_lua	access_log.lua
SystemMonitor（部分）	lua-resty-prometheus	monitor.lua
PortalServlet	content_by_lua	portal.lua
10. 参考文档
《TSG 7.6 架构迁移确认书 v1.0》

《中宇万通安全网关技术白皮书》2019.10

OpenResty 官方文档

Tongsuo 国密 OpenSSL 使用指南

设计批准：________________ 日期：________

text

请复制以上全部内容，保存为 `TrustMore_架构升级_概要设计.md`，即可直接使用。
本回答由 AI 生成，内容仅供参考，请仔细甄别。
