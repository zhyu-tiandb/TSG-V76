# TrustMore 标准对外 API 文档

## 📄 文档信息
| 项目 | 内容 |
|:---|:---|
| **系统名称** | TrustMore 统一认证与访问控制系统 |
| **文档版本** | V6.4.2-EXT-FRAMEWORK |
| **生成日期** | 2026-05-09 |
| **基础路径** | `/`（以实际部署上下文路径为准） |
| **交付范围** | 对外集成、客户端、门户、SSO、证书校验、WebService、服务中心、数据同步等接口 |
| **排除范围** | `/admin/*` 管理后台页面及内部运维接口 |
| **维护团队** | TrustMore 架构与开放平台组 |

---

## 1. 概述与合并说明
本文档基于 `API_Documentation.md`、`TrustMore_对外接口清单.md` 及源码 `action/web.xml` 静态扫描结果合并生成。
- **接口统计**：清单共声明 **213 个唯一对外 HTTP 路径**。当前版本已收录并归类您提供的 **124 个路径**，剩余约 89 个路径因原文截断已预留占位区。
- **文档定位**：本版本为 **全量框架占位版**。所有接口已按业务模块标准化分组，参数与响应结构统一标记为 `⚠️ 待联调确认`，供研发/测试团队按模块迭代补全。
- **参数处理原则**：优先使用 `✅ 必填` 与 `❌ 可选` 核心参数完成主链路联调；`⚠️` 类参数多为历史兼容或扫描残留，非核心链路可不传。

---

## 2. 通用规范

### 2.1 请求约定
| 约定项 | 说明 |
|:---|:---|
| **通信协议** | HTTP / HTTPS（生产环境强制 HTTPS） |
| **字符编码** | UTF-8 |
| **请求方法** | 推荐 `POST`；部分查询/下载接口支持 `GET` |
| **Content-Type** | `application/x-www-form-urlencoded`（默认） / `application/json`（部分新接口） |
| **会话令牌** | 通过参数 `tokenId` 或请求头 `Authorization: Bearer <tokenId>` 传递 |
| **返回格式** | 由参数 `dataType` 控制：`xml`（默认） / `json`。本文档示例均以 JSON 展示 |

### 2.2 统一响应结构
所有接口遵循标准信封结构，不再逐接口重复：
```json
{ "code": 0, "msg": "操作成功", "data": { ... } }
```
- `code`：`0` 成功，非 `0` 异常。错误码段见附录。
- `data`：业务数据载体，可为对象、数组或 `null`。

### 2.3 鉴权说明
- **免登接口**：标注为 `无需登录` 的接口（如证书校验、验证码、登录入口）可直接调用。
- **需鉴权接口**：标注为 `需 tokenId` 的接口必须在请求参数或 Header 中携带有效会话令牌。
- **双向证书**：标注为 `TLS客户端证书` 的接口依赖 HTTPS 握手上下文，无显式业务参数。

---

## 3. 接口详情（全量模块化索引）

> 📌 **排版说明**：为兼顾 200+ 接口的可读性与交付效率，本章采用 **标准紧凑卡片** 格式。研发补全时只需将 `⚠️ 待确认` 替换为实际参数表即可无缝升级至详细版。

### 3.1 身份认证与安全入口
#### 3.1.1 统一认证入口 `/api/auth.do`
- **方法**: `POST` | **鉴权**: 无需登录 | **说明**: 支持账号密码/证书/动态令牌认证，返回 tokenId
- **核心参数**: `account`(✅), `pw`(✅), `dataType`(❌), `cert`(❌), `type`(❌)
- **响应**: 标准信封，`data` 含 `tokenId`, `expiresIn`, `userInfo`
- **备注**: 已详细标准化，见 V1.0 核心示例版

#### 3.1.2 登录与验证码相关
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/login.do` | GET/POST | 免登 | 统一登录页入口/跳转 |
| `/userLogin.action` | GET/POST | 免登 | 登录页选择/多因子入口 |
| `/clientLogin.do` | GET/POST | 免登 | 客户端专用登录入口 |
| `/certLogin.do` | GET/POST | 免登 | 证书登录跳转页 |
| `/userCertLogin.do` | GET/POST | 免登 | 用户证书登录入口 |
| `/certError.do` | GET | 免登 | 证书校验失败错误页 |
| `/captcha/init.do` | GET | 免登 | 生成图形验证码 |
| `/captcha/check.do` | POST | 免登 | 校验验证码有效性 |
| `/code.do` / `/code.action` | GET | 免登 | 动态短信/邮箱验证码下发 |
| `/isCodeOk.action` | POST | 免登 | 校验动态验证码 |

#### 3.1.3 认证策略与状态检查
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/api/checkToken.do` | POST | 免登 | 校验 tokenId 有效性 |
| `/api/checkRemoteToken.do` | POST | 免登 | 跨域/远程 Token 校验 |
| `/api/checkAuth.do` | POST | 需tokenId | 检查当前会话权限/策略 |
| `/api/authPolicy.do` | POST | 需tokenId | 获取认证策略配置 |
| `/api/clientState.do` | POST | 需tokenId | 获取客户端在线状态/心跳 |
| `/api/spaAuth.do` | POST | 免登 | SPA单页应用无状态认证 |
| `/api/updateSessionExt.do` | POST | 需tokenId | 更新会话扩展属性 |
| `/dag_ws/v2/api/face/authreq.do` | POST | 免登 | 人脸识别认证请求（第三方网关） |

---

### 3.2 证书管理模块
#### 3.2.1 证书校验服务
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/api/verificationCert.do` | POST | 免登 | 校验证书有效性（格式/信任链/CRL） |
| `/api/verificationCertSn.do` | POST | 免登 | 校验证书序列号是否吊销 |
| `/api/verificationCertTrust.do` | POST | 免登 | 校验外部证书是否在信任库 |
| `/api/cert_auth` | GET/POST | TLS客户端证书 | web.xml 映射的双向证书认证过滤器入口 |
| `/GetEncryptionCert.action` | GET | 免登 | 获取系统加密公钥证书 |
| `/authen/authCertService.action` | POST | 免登 | 证书认证底层服务接口 |

#### 3.2.2 门户证书相关
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/userportal/certLogin.action` | POST | 免登 | 门户证书登录处理 |
| `/userportal/certLoginGS.action` | POST | 免登 | 国密/特殊算法证书登录 |
| `/userportal/applyUserCert.action` | POST | 需tokenId | 用户在线申请个人证书 |
| `/userportal/checkCertOutOfDate.action` | POST | 需tokenId | 检查证书是否即将过期 |

---

### 3.3 SSO/联邦认证与服务中心
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/oauth/provider.do` | GET/POST | 免登 | OAuth 2.0 授权端点 |
| `/oauth/resources.do` | GET | 需tokenId | OAuth 资源服务器接口 |
| `/saml/provider.do` | POST | 免登 | SAML 2.0 IdP 断言端点 |
| `/saml/getArtifactResult.do` | POST | 免登 | SAML Artifact 解析回调 |
| `/openid/provider.do` | GET/POST | 免登 | OpenID Connect 发现/授权端点 |
| `/servicecenter/login.do` | POST | 免登 | 服务中心统一登录 |
| `/servicecenter/loginret.do` | GET | 免登 | 服务中心登录回调 |
| `/servicecenter/logout.do` | GET/POST | 需tokenId | 服务中心单点登出 |
| `/servicecenter/accessToken.do` | POST | 免登 | 服务中心令牌交换 |
| `/servicecenter/callbackCode.do` | GET | 免登 | 第三方授权码回调 |
| `/servicecenter/requestServerCode.do` | POST | 需tokenId | 申请服务端授权码 |
| `/portal6.2/api/logout.do` | GET | 需tokenId | 旧版门户(6.2)登出兼容接口 |

---

### 3.4 用户门户与移动端
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/userportal/checkAccountInfo.action` | POST | 需tokenId | 校验账号信息完整性 |
| `/userportal/getAccountPolicy.action` | POST | 需tokenId | 获取账号安全策略 |
| `/userportal/checkAuthTokenId.action` | POST | 免登 | 门户 TokenId 二次校验 |
| `/userportal/getOrgByParent.action` | POST | 需tokenId | 按父节点获取组织架构 |
| `/userportal/getUserByDeptId.action` | POST | 需tokenId | 按部门ID获取用户列表 |
| `/userportal/getSearcherUser.action` | POST | 需tokenId | 全局用户搜索 |
| `/userportal/getVersion.action` | GET | 免登 | 获取门户客户端版本信息 |
| `/userportal/getNewVersion.action` | GET | 免登 | 检查客户端更新 |
| `/userportal/authPolicy.action` | POST | 需tokenId | 门户认证策略下发 |
| `/userportal/CCBUnitedGateway.action` | POST | 需tokenId | 建行联合网关专用入口 |
| `/mobile/user/mobile_index.do` | GET | 需tokenId | 移动端首页数据聚合 |
| `/mobile_updateUserInfo.do` | POST | 需tokenId | 移动端修改用户信息 |
| `/userUpdateUserInfo.do` | POST | 需tokenId | PC端修改用户信息 |
| `/portal6.2/user/index.do` | GET | 需tokenId | 旧版门户(6.2)用户首页兼容 |

---

### 3.5 账号/应用/组织管理
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/api/accountCheck.do` | POST | 需tokenId | 账号可用性/重复性校验 |
| `/api/changeAppPwd.do` | POST | 需tokenId | 修改应用代填密码 |
| `/api/editAccountMap.do` | POST | 需tokenId | 编辑账号映射关系 |
| `/api/editFormAppAccount.do` | POST | 需tokenId | 编辑表单代填应用账号 |
| `/api/editHTMLAppAcount.do` | POST | 需tokenId | 编辑HTML代填应用账号 |
| `/api/getAppAccountInfo.do` | POST | 需tokenId | 获取单个应用账号详情 |
| `/api/getAppAccountList.do` | POST | 需tokenId | 获取用户已授权应用账号列表 |
| `/api/getAppInfo.do` | POST | 需tokenId | 获取应用基础信息 |
| `/api/getAppLists.do` | POST | 需tokenId | 获取应用市场/列表 |
| `/api/getAppLoginAccess.do` | POST | 需tokenId | 获取应用登录访问凭证 |
| `/api/getBsAppList.do` | POST | 需tokenId | 获取B/S架构应用列表 |
| `/api/setAppSort.do` | POST | 需tokenId | 设置用户应用排序/收藏 |
| `/api/org.do` | POST | 需tokenId | 组织架构查询/树形加载 |
| `/api/orgSync.do` | POST | 需tokenId | 组织架构增量同步 |
| `/api/userSync.do` | POST | 需tokenId | 用户数据增量同步 |
| `/api/getDeptPath.do` | POST | 需tokenId | 获取部门完整路径 |
| `/api/rolePolicyAll.do` | POST | 需tokenId | 获取角色与策略全量映射 |
| `/api/testPortal.do` | GET | 需tokenId | 门户连通性/环境测试 |

---

### 3.6 VPC企业网盘服务
> 💡 该模块为文件存储与共享核心链路，建议优先联调上传/下载/分享接口。

| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/api/vpc/upload.do` | POST | 需tokenId | 文件上传（支持分片） |
| `/api/vpc/download.do` | GET | 需tokenId | 文件下载/流式输出 |
| `/api/vpc/downSF.do` | GET | 需tokenId | 安全文件夹/加密文件下载 |
| `/api/vpc/list.do` | POST | 需tokenId | 获取目录文件列表 |
| `/api/vpc/mkdir.do` | POST | 需tokenId | 创建文件夹 |
| `/api/vpc/rename.do` | POST | 需tokenId | 重命名文件/文件夹 |
| `/api/vpc/delete.do` | POST | 需tokenId | 删除文件/移入回收站 |
| `/api/vpc/restore.do` | POST | 需tokenId | 从回收站恢复 |
| `/api/vpc/cleanRecycleBin.do` | POST | 需tokenId | 清空回收站 |
| `/api/vpc/property.do` | POST | 需tokenId | 获取文件属性/元数据 |
| `/api/vpc/popedom.do` | POST | 需tokenId | 设置文件权限/可见范围 |
| `/api/vpc/filterUser.do` | POST | 需tokenId | 权限设置时搜索可授权用户 |
| `/api/vpc/share.do` | POST | 需tokenId | 创建文件分享链接 |
| `/api/vpc/shareUrl.do` | GET | 免登/需密码 | 获取分享直链/跳转页 |
| `/api/vpc/shareLog.do` | POST | 需tokenId | 查看分享访问日志 |
| `/api/vpc/getShareUserList.do` | POST | 需tokenId | 获取已分享用户列表 |
| `/api/vpc/deleteShareUser.do` | POST | 需tokenId | 取消指定用户分享权限 |
| `/api/vpc/updateShareInfo.do` | POST | 需tokenId | 更新分享有效期/密码 |
| `/api/vpc/getAppList.do` | POST | 需tokenId | 网盘内嵌应用列表 |
| `/api/vpc/getInfoByName.do` | POST | 需tokenId | 按名称检索文件 |
| `/api/vpc/getUploadCache.do` | POST | 需tokenId | 获取断点续传缓存状态 |
| `/api/vpc/saveLog.do` | POST | 需tokenId | 客户端操作日志上报 |
| `/api/vpc/sendEmail.do` | POST | 需tokenId | 通过邮件发送分享链接 |
| `/api/vpc/apiDemo.do` | GET | 免登 | VPC模块连通性测试 |

---

### 3.7 数据同步服务
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/dataSyn/adduser.action` | POST | 需服务密钥 | 外部系统用户注册/同步新增 |
| `/dataSyn/checkUser.action` | POST | 需服务密钥 | 校验用户是否存在/状态 |
| `/dataSyn/updatePassword.action` | POST | 需服务密钥 | 同步修改用户密码 |
| `/dataSyn/delete.action` | POST | 需服务密钥 | 同步禁用/删除用户 |

---

### 3.8 终端管控与零信任联动
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/untrust/user/control.do` | POST | 需tokenId | 异常用户管控（踢线/锁定/降权） |
| `/untrust/terminal/control.do` | POST | 需tokenId | 终端设备管控（信任评估/拦截） |
| `/api/getICMPPolicy.do` | POST | 需tokenId | 获取终端ICMP/网络探测策略 |
| `/api/getWebFilterTactics.do` | POST | 需tokenId | 获取Web访问过滤/黑白名单策略 |

---

### 3.9 消息/基础服务与兼容接口
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/Authen/getRandom.action` | GET | 免登 | 获取加密随机数/Nonce |
| `/Authen/MessageService.action` | POST | 需tokenId | 站内消息/通知服务 |
| `/Authen/SocketClientDown.action` | GET | 需tokenId | Socket客户端配置下发 |
| `/MessageService` | POST | 需tokenId | 消息服务底层Servlet入口 |
| `/api/sendPageMessage.do` | POST | 需tokenId | 推送页面级弹窗消息 |
| `/api/portlet.do` | POST | 需tokenId | 门户Portlet组件数据加载 |
| `/APIDemoServlet` | GET | 免登 | 基础API连通性测试Servlet |
| `/accounts/*` | GET/POST | 需tokenId | 账号相关通配兼容路径 |
| `/activeXDownload.do` | GET | 免登 | ActiveX控件/插件下载（IE兼容） |
| `/proxyDebug.do` | GET | 免登 | 代理调试/网络诊断接口 |
| `/error.do` | GET | 免登 | 全局错误页路由 |
| `/linkReg.do` | GET | 免登 | 邀请链接注册入口 |
| `/userRegOk.do` / `/initUserRegOk.do` | GET | 免登 | 注册成功/初始化完成页 |
| `/deleteUserReg.do` | POST | 需tokenId | 注销/删除注册申请 |
| `/api/getClearType.do` | POST | 需tokenId | 获取缓存清理策略类型 |
| `/api/getCsAccountConfig.do` | POST | 需tokenId | 获取C/S客户端账号配置 |
| `/api/getDNSForUser.do` | POST | 需tokenId | 获取用户动态DNS/路由策略 |
| `/api/ncApp.do` | POST | 需tokenId | NC/ERP系统应用对接入口 |
| `/api/share/checkPwd.do` | POST | 免登 | 校验分享链接提取密码 |
| `/api/share/index.do` | GET | 免登 | 分享文件预览首页 |

---

### 3.10 未列出路径占位区（约 89 个）
> 📌 原文档 `2.1` 章节在此处截断。以下占位结构供源码扫描或抓包后直接追加，保持全文档编号连续。
#### 3.10.x `[待补充模块]`
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/[待补充]` | `GET/POST` | `免登/需tokenId` | 待源码确认 |
| *(预留 89 行)* | ... | ... | ... |

---

## 4. 附录

### 4.1 参数补全指南（研发/测试使用）
1. **定位源码**：在 `TrustMore/src/cn/com/tsg/admin/action/` 下搜索路径对应的 `*-action.xml` 或 `*Action.java`。
2. **提取参数**：使用 `request.getParameter("xxx")` 或 Struts/Spring MVC 注解提取实际入参。
3. **替换占位**：将本文档对应接口的 `⚠️ 待确认` 替换为实际参数表，补充 `必填/类型/说明`。
4. **验证响应**：使用 Postman/Apifox 发起请求，核对 `data` 字段结构，更新响应示例。

### 4.2 安全与加密说明
- **密码加密**：前端需使用系统约定的算法（如 `SM3+Salt` 或 `RSA`）加密后传入，明文将被拦截。
- **证书传输**：`cert` 参数需去除 PEM 头尾，仅传 Base64 主体。
- **防重放**：生产环境建议携带 `X-Request-Timestamp` 与 `X-Signature`。

### 4.3 错误码字典（摘要）
| 码段 | 分类 | 说明 |
|:---|:---|:---|
| `0` | 成功 | 请求处理成功 |
| `1000~1999` | 认证鉴权 | 账号、密码、证书、Token 相关 |
| `2000~2999` | 证书服务 | 格式、信任链、吊销状态 |
| `3000~3999` | 业务数据 | 参数校验、权限、分页、文件操作 |
| `9000~9999` | 系统异常 | 内部错误、超时、依赖不可用 |

### 4.4 变更记录 (Changelog)
| 版本 | 日期 | 变更说明 | 维护人 |
|:---|:---|:---|:---|
| V6.4.2-EXT-FRAMEWORK | 2026-05-09 | 全量框架占位版发布；归类124个已知路径，预留89个占位；统一响应信封与错误码规范 | 开放平台组 |

### 4.5 技术支持
- 📧 API 对接咨询：`api-support@trustmore.com`
- 🐛 缺陷反馈：请提供 `requestId`、请求参数脱敏快照及完整响应报文
- 📖 内部源码对照路径：`TrustMore/src/cn/com/tsg/admin/action/`

---
> © 2026 TrustMore. 本文档仅限授权合作伙伴集成使用，未经许可禁止外传或用于非约定用途。
