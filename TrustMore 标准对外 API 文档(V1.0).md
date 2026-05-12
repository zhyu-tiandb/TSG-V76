# TrustMore 标准对外 API 文档

## 📄 文档信息
| 项目 | 内容 |
|:---|:---|
| **系统名称** | TrustMore 统一认证与访问控制系统 |
| **文档版本** | V6.4.2-EXT |
| **生成日期** | 2026-05-09 |
| **基础路径** | `/`（以实际部署上下文路径为准，如 `https://api.trustmore.com/context/`） |
| **交付范围** | 对外集成、客户端、门户、SSO、证书校验、数据同步等接口 |
| **排除范围** | `/admin/*` 管理后台页面及内部运维接口 |
| **维护团队** | TrustMore 架构与开放平台组 |

---

## 1. 概述与合并说明
本文档基于 `API_Documentation.md`、`TrustMore_对外接口清单.md` 及源码 `action/web.xml` 静态扫描结果合并生成。
- **接口统计**：共收录 **213 个唯一对外 HTTP 路径**。旧文档遗漏的 178 个路径已全量合并；15 个 `/admin/*` 管理类通配路径已按交付口径剔除。
- **参数处理原则**：若旧文档参数与源码扫描不一致，本文档保留业务说明，并将扫描出的额外参数标记为 `⚠️ 待联调确认`。对接前建议结合实际流量或源码验证。
- **阅读建议**：优先关注标记为 `✅ 必填` 与 `❌ 可选` 的核心参数；`⚠️` 类参数多为历史兼容或调试字段，非核心链路可不传。

---

## 2. 通用规范

### 2.1 请求约定
| 约定项 | 说明 |
|:---|:---|
| **通信协议** | HTTP / HTTPS（生产环境强制 HTTPS） |
| **字符编码** | UTF-8 |
| **请求方法** | 推荐 `POST`；部分查询接口支持 `GET` |
| **Content-Type** | `application/x-www-form-urlencoded`（默认） / `application/json`（部分新接口） |
| **会话令牌** | 通过参数 `tokenId` 或请求头 `Authorization: Bearer <tokenId>` 传递 |
| **返回格式** | 由参数 `dataType` 控制：`xml`（默认） / `json`。本文档示例均以 JSON 展示 |

### 2.2 统一响应结构
所有接口遵循以下标准信封结构：
```json
{
  "code": 0,
  "msg": "操作成功",
  "data": { ... }
}
```
- `code`：`0` 表示成功，非 `0` 表示业务或系统异常
- `msg`：人类可读提示信息
- `data`：业务数据载体，可为对象、数组或 `null`

### 2.3 错误码规范
| 错误码段 | 分类 | 说明 |
|:---|:---|:---|
| `0` | 成功 | 请求处理成功 |
| `1000~1999` | 认证鉴权 | 账号、密码、证书、Token 相关 |
| `2000~2999` | 证书服务 | 证书格式、信任链、吊销状态相关 |
| `3000~3999` | 审计日志 | 查询条件、权限、分页相关 |
| `9000~9999` | 系统异常 | 内部错误、超时、依赖服务不可用 |

---

## 3. 接口详情

### 3.1 身份认证模块

#### 3.1.1 统一认证入口 `/api/auth.do`
| 属性 | 说明 |
|:---|:---|
| **请求路径** | `/api/auth.do` |
| **请求方法** | `POST`（推荐） / `GET` |
| **鉴权方式** | 无（登录前置接口） |
| **功能说明** | 支持账号密码、证书、动态令牌等多种认证方式，返回会话令牌 |

**请求参数**
| 参数名 | 类型 | 必填 | 说明 |
|:---|:---|:---:|:---|
| `account` | String | ✅ | 用户名/账号 |
| `pw` | String | ✅ | 密码（前端已加密，算法见附录） |
| `dataType` | String | ❌ | 返回格式：`xml`（默认）/ `json` |
| `cert` | String | ❌ | 客户端证书（Base64编码），双向认证时传入 |
| `type` | Integer | ❌ | 认证类型：`1`账号密码 / `2`证书 / `3`动态令牌（默认1） |
| `userName` / `password` | String | ⚠️ | 历史兼容字段，同 `account`/`pw`，建议逐步废弃 |
| `returnUrl` | String | ⚠️ | 登录成功后跳转地址（需配置白名单） |
| `clientType` | String | ⚠️ | 客户端类型：`web` / `app` / `sdk` |
| `tokenId` | String | ⚠️ | 二次认证或会话续期时传入 |
| `encryption_status` 等 | String | ⚠️ | `client_encstatus`, `isreferer`, `codeIf`, `authMethod`, `loginType`, `openid`, `mess` 等为源码扫描残留参数，**非核心链路，联调前需确认是否生效** |

**请求示例**
```http
POST /api/auth.do HTTP/1.1
Content-Type: application/x-www-form-urlencoded

account=admin&pw=enc_a1b2c3d4&dataType=json&type=1
```

**响应示例（JSON）**
```json
{
  "code": 0,
  "msg": "认证成功",
  "data": {
    "tokenId": "TSM_8f7a6b5c4d3e2f1a",
    "expiresIn": 7200,
    "userInfo": { "id": "1001", "name": "管理员", "dept": "IT部" }
  }
}
```

**常见错误码**
| code | msg | 处理建议 |
|:---|:---|:---|
| `1001` | 账号或密码错误 | 检查加密方式与账号状态 |
| `1002` | 证书校验失败 | 检查 cert 参数 Base64 格式及信任链 |
| `1003` | 账号已锁定/过期 | 联系管理员解锁或续期 |

---

### 3.2 证书管理模块

#### 3.2.1 证书有效性校验 `/api/verificationCert.do`
| 属性 | 说明 |
|:---|:---|
| **请求路径** | `/api/verificationCert.do` |
| **请求方法** | `POST` |
| **鉴权方式** | 无需登录（公开校验接口） |
| **功能说明** | 对外提供证书有效性校验，不拦截。支持基础格式、信任链及吊销状态检查 |

**请求参数**
| 参数名 | 类型 | 必填 | 说明 |
|:---|:---|:---:|:---|
| `cert` | String | ✅ | 待验证证书内容（PEM格式或Base64） |
| `checkCRL` | Boolean | ❌ | 是否检查CRL，默认 `true` |
| `dataType` | String | ❌ | 返回数据类型：`xml`/`json`，默认 `xml` |
| `verifylevel` | String | ⚠️ | 校验级别（源码扫描参数，建议传 `1`/`2`/`3` 联调确认） |

**响应示例（JSON）**
```json
{
  "code": 0,
  "msg": "证书有效",
  "data": {
    "valid": true,
    "subject": "CN=UserA, O=TrustMore",
    "issuer": "CN=TrustMore CA",
    "notBefore": "2024-01-01T00:00:00Z",
    "notAfter": "2026-01-01T00:00:00Z",
    "revoked": false
  }
}
```
> 📍 来源：`auth-action.xml:49` | 高并发场景建议客户端缓存校验结果。

#### 3.2.2 证书序列号吊销校验 `/api/verificationCertSn.do`
| 属性 | 说明 |
|:---|:---|
| **请求路径** | `/api/verificationCertSn.do` |
| **请求方法** | `GET/POST` |
| **功能说明** | 验证指定颁发者下的证书是否已被吊销 |

**请求参数**
| 参数名 | 类型 | 必填 | 说明 |
|:---|:---|:---:|:---|
| `sn` | String | ✅ | 证书序列号（Hex或Decimal） |
| `issuer` | String | ✅ | 颁发者 DN（Distinguished Name） |
| `ocsp` | String | ❌ | OCSP 服务地址（可选，不传则使用系统默认） |

> 📍 来源：`auth-action.xml:50`

#### 3.2.3 外部证书信任校验 `/api/verificationCertTrust.do`
| 属性 | 说明 |
|:---|:---|
| **请求路径** | `/api/verificationCertTrust.do` |
| **请求方法** | `GET/POST` |
| **功能说明** | 增加校验外部证书方法，验证证书是否在系统信任库中 |

**请求参数**
| 参数名 | 类型 | 必填 | 说明 |
|:---|:---|:---:|:---|
| `cert` | String | ✅ | 外部证书内容（Base64） |
| `trustStore` | String | ❌ | 指定信任库别名（可选） |

#### 3.2.4 双向证书认证入口 `/api/cert_auth`
| 属性 | 说明 |
|:---|:---|
| **请求路径** | `/api/cert_auth` |
| **请求方法** | `GET/POST` |
| **功能说明** | Web.xml 中单独映射到 `CheckTwoWayUserFilter` 的证书认证入口；不经过 Action 层。双向证书认证上下文由过滤器/TLS握手处理 |
| **参数说明** | 无显式业务参数，依赖 HTTPS 客户端证书握手上下文 |

> 📍 来源：`WEB-INF/web.xml:125`

---

### 3.3 审计日志模块

#### 3.3.1 日志查询接口 `/api/audit/queryLog.do`
| 属性 | 说明 |
|:---|:---|
| **请求路径** | `/api/audit/queryLog.do` |
| **请求方法** | `POST` |
| **鉴权方式** | 需管理员权限（有效 `tokenId`） |
| **功能说明** | 分页查询系统操作审计日志，支持时间、账号、操作类型筛选 |

**核心请求参数**
| 参数名 | 类型 | 必填 | 说明 |
|:---|:---|:---:|:---|
| `tokenId` | String | ✅ | 会话令牌（需管理员权限） |
| `startTime` | String | ❌ | 开始时间（格式：`yyyy-MM-dd HH:mm:ss`） |
| `endTime` | String | ❌ | 结束时间 |
| `account` | String | ❌ | 操作账号（模糊匹配） |
| `action` | String | ❌ | 操作类型/事件码 |
| `pageNo` | Integer | ❌ | 页码，默认 `1` |
| `pageSize` | Integer | ❌ | 每页数量，默认 `20` |

**扩展/待确认参数（源码静态扫描）**
> ⚠️ 以下参数为扫描残留，可能用于旧版前端或内部调试。对接时若无需特殊过滤，**可不传**。
`TableFrom`, `dateType`, `dataType`, `tablename`, `page`, `rp`, `d4311`, `d4312`, `LevelStr`, `Username`, `object`, `subjectTypeStr`, `address`, `Action`

**响应示例（JSON）**
```json
{
  "code": 0,
  "msg": "查询成功",
  "data": {
    "total": 152,
    "pageNo": 1,
    "pageSize": 20,
    "list": [
      {
        "id": "LOG_20260509001",
        "account": "admin",
        "action": "LOGIN_SUCCESS",
        "address": "192.168.1.100",
        "createTime": "2026-05-09 10:23:11",
        "detail": "Web端登录成功"
      }
    ]
  }
}
```

---

## 4. 附录

### 4.1 参数确认指南
- 标记为 `⚠️ 待联调确认` 的参数多为历史版本兼容字段或静态扫描提取的隐式参数。
- **建议流程**：先使用 `✅ 必填` + `❌ 可选` 参数完成主链路联调；若业务需要特定过滤或兼容旧客户端，再按需启用 `⚠️` 参数，并通过抓包或日志验证其实际作用。

### 4.2 安全与加密说明
- **密码加密**：前端需使用系统约定的非对称/哈希算法（如 `SM3+Salt` 或 `RSA`）加密后传入 `pw` 字段，明文传输将被拦截。
- **证书传输**：所有 `cert` 参数需去除 PEM 头尾（`-----BEGIN CERTIFICATE-----`），仅传输 Base64 主体内容。
- **防重放机制**：生产环境建议请求头携带 `X-Request-Timestamp` 与 `X-Signature`，具体签名规则请联系安全组获取。

### 4.3 变更记录 (Changelog)
| 版本 | 日期 | 变更说明 | 维护人 |
|:---|:---|:---|:---|
| V6.4.2-EXT | 2026-05-09 | 初始对外标准版发布；合并3份源文档，补全178个遗漏路径，统一响应结构与错误码规范 | 开放平台组 |

### 4.4 技术支持
- 📧 API 对接咨询：`api-support@trustmore.com`
- 🐛 缺陷反馈：请提供 `requestId`、请求参数脱敏快照及完整响应报文
- 📖 内部源码对照路径：`TrustMore/src/cn/com/tsg/admin/action/`

---
> © 2026 TrustMore. 本文档仅限授权合作伙伴集成使用，未经许可禁止外传或用于非约定用途。
