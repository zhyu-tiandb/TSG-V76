# TrustMore 标准对外 API 文档（V1.2）

## 文档信息

| 项目 | 内容 |
|:---|:---|
| 系统名称 | TrustMore 统一认证与访问控制系统 |
| 系统版本 | 6.4.2 |
| 文档版本 | V1.2 |
| 生成日期 | 2026-05-09 |
| 基础路径 | `/`，以实际部署上下文路径为准 |
| 合并来源 | `TrustMore_标准对外API文档.md`、`#TrustMore 标准对外 API 文档(V1.1).md`、`TrustMore/API_Documentation.md`、源码 action/web.xml 扫描结果 |
| 交付范围 | 对外集成、客户端、门户、SSO、证书校验、WebService、服务中心、数据同步等接口 |
| 排除范围 | `/admin/*` 管理后台页面及内部运维接口 |
| 维护团队 | TrustMore 架构与开放平台组 |

## 1. 概述与整合说明

本文档将 `TrustMore_标准对外API文档.md` 与 `#TrustMore 标准对外 API 文档(V1.1).md` 统一为 V1.2。V1.1 中的“未列出路径占位区”已由源码扫描得到的全量接口清单补齐，形成可对外提供的标准 API Markdown 文档。

- 对外 HTTP 入口：215 条。
- 唯一对外 HTTP 路径：213 个。
- HTTP 接口详细定义：215 条，包含重复配置的 `/api/auth.do`、`/api/userInfo.do`。
- WebService / SOAP 接口：保留 TrustMore 内置 WebService、Coremail API 以及 CMCC 相关 JAX-WS 接口。
- V1.1 中约 89 个占位接口已补齐；不再保留占位区。

### 1.1 合并与遗漏分析

- `API_Documentation.md` 识别到唯一接口/通配路径：50 个。
- `TrustMore_对外接口清单.md` 识别到唯一对外 HTTP 路径：213 个。
- 旧文档相对对外清单遗漏的具体对外路径：178 个，已全部合并到本文档。
- 旧文档中未进入本文档的路径/通配符：15 个，主要是 `/admin/*` 管理后台接口或通配前缀，不作为标准对外接口交付。
- 若旧文档参数与源码扫描参数不一致，本文档保留旧文档说明，同时在参数表中补充源码扫描出的参数，并标记为“源码未判定”。

## 2. 通用规范

### 2.1 请求约定

| 约定项 | 说明 |
|:---|:---|
| 通信协议 | HTTP / HTTPS，生产环境建议强制 HTTPS |
| 字符编码 | UTF-8 |
| 请求方法 | 旧 action 框架多数接口同时接受 GET/POST；对外集成建议优先使用 POST，下载/页面跳转类接口可使用 GET |
| Content-Type | `application/x-www-form-urlencoded` 为主；文件上传使用 `multipart/form-data`；少数接口可按实现传 JSON |
| 会话令牌 | 常见参数为 `tokenId`；部分接口也可能从 Header 或 Session 中读取认证上下文 |
| 返回格式 | 由接口实现和 `dataType/dateType` 参数决定，常见为 XML、JSON、HTML 跳转、文本或文件流 |
| WebService 地址 | `webservice_path` 配置项；默认 `http://localhost:8800/webserviceapi` |

### 2.2 鉴权说明

- 登录、验证码、证书校验、SSO Provider、部分分享入口可免登录调用。
- 业务查询、策略下发、账号配置、文件操作等接口通常需要有效 `tokenId`。
- `/api/cert_auth` 依赖双向 TLS 客户端证书上下文，由过滤器和 URL handler 处理。
- `dataSyn/*`、`servicecenter/*` 等系统对接接口可能还依赖服务端密钥、回调码或外部系统白名单，需联调确认。

### 2.3 响应约定

旧接口返回结构不完全统一。对外交付时建议按以下信封理解和适配，实际返回以接口实现为准：

```json
{
  "code": "100",
  "message": "操作成功",
  "data": {}
}
```

## 3. 接口分类统计

| 分类 | 接口数 |
|---|---:|
| REST API | 65 |
| 文件/云盘接口 | 26 |
| 证书接口 | 21 |
| 标准门户接口 | 22 |
| 用户验证/管理接口 | 37 |
| SSO/协议接口 | 6 |
| 消息认证接口 | 4 |
| 服务中心接口 | 6 |
| 数据同步接口 | 4 |
| 零信任联动接口 | 2 |
| 验证码接口 | 2 |
| 门户兼容入口 | 3 |
| 登录/兼容入口 | 17 |

## 4. V1.1 模块化索引（已补齐）

本节保留 V1.1 的模块化交付视图，用于快速查找主要业务接口；完整参数和源码位置见第 7 章。


### 5.1 身份认证与安全入口
#### 5.1.1 统一认证入口 `/api/auth.do`
- **方法**: `POST` | **鉴权**: 无需登录 | **说明**: 支持账号密码/证书/动态令牌认证，返回 tokenId
- **核心参数**: `account`(✅), `pw`(✅), `dataType`(❌), `cert`(❌), `type`(❌)
- **响应**: 标准信封，`data` 含 `tokenId`, `expiresIn`, `userInfo`
- **备注**: 已详细标准化，见 V1.0 核心示例版

#### 5.1.2 登录与验证码相关
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

#### 5.1.3 认证策略与状态检查
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

### 5.2 证书管理模块
#### 5.2.1 证书校验服务
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/api/verificationCert.do` | POST | 免登 | 校验证书有效性（格式/信任链/CRL） |
| `/api/verificationCertSn.do` | POST | 免登 | 校验证书序列号是否吊销 |
| `/api/verificationCertTrust.do` | POST | 免登 | 校验外部证书是否在信任库 |
| `/api/cert_auth` | GET/POST | TLS客户端证书 | web.xml 映射的双向证书认证过滤器入口 |
| `/GetEncryptionCert.action` | GET | 免登 | 获取系统加密公钥证书 |
| `/authen/authCertService.action` | POST | 免登 | 证书认证底层服务接口 |

#### 5.2.2 门户证书相关
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/userportal/certLogin.action` | POST | 免登 | 门户证书登录处理 |
| `/userportal/certLoginGS.action` | POST | 免登 | 国密/特殊算法证书登录 |
| `/userportal/applyUserCert.action` | POST | 需tokenId | 用户在线申请个人证书 |
| `/userportal/checkCertOutOfDate.action` | POST | 需tokenId | 检查证书是否即将过期 |

---

### 5.3 SSO/联邦认证与服务中心
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

### 5.4 用户门户与移动端
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

### 5.5 账号/应用/组织管理
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

### 5.6 VPC企业网盘服务
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

### 5.7 数据同步服务
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/dataSyn/adduser.action` | POST | 需服务密钥 | 外部系统用户注册/同步新增 |
| `/dataSyn/checkUser.action` | POST | 需服务密钥 | 校验用户是否存在/状态 |
| `/dataSyn/updatePassword.action` | POST | 需服务密钥 | 同步修改用户密码 |
| `/dataSyn/delete.action` | POST | 需服务密钥 | 同步禁用/删除用户 |

---

### 5.8 终端管控与零信任联动
| 路径 | 方法 | 鉴权 | 功能说明 |
|:---|:---|:---|:---|
| `/untrust/user/control.do` | POST | 需tokenId | 异常用户管控（踢线/锁定/降权） |
| `/untrust/terminal/control.do` | POST | 需tokenId | 终端设备管控（信任评估/拦截） |
| `/api/getICMPPolicy.do` | POST | 需tokenId | 获取终端ICMP/网络探测策略 |
| `/api/getWebFilterTactics.do` | POST | 需tokenId | 获取Web访问过滤/黑白名单策略 |

---

### 5.9 消息/基础服务与兼容接口
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

## 5. 全量接口总览

### 5.1 REST API

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/api/accountCheck.do` | GET/POST | @return 1 成功；0失败。 | `userName, passWord` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:97` |
| `/api/adLogin.do` | GET/POST | 通过Active Directory进行域认证 | `uid` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:28` |
| `/api/appAccountConfig.do` | GET/POST | 获得应用列表 | `userName, password, appId, type` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:116` |
| `/api/appAuth.do` | GET/POST | 应用帐号认证 | `userName, password, dataType, returnUrl, appCode` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:87` |
| `/api/audit/queryLog.do` | GET/POST | 获得日志列表 | `TableFrom, dateType, tokenId, dataType, tablename, page, rp, d4311, d4312, LevelStr, Username, object, subjectTypeStr, address, Action, ObjectType, Result` | `TrustMore/src/cn/com/tsg/admin/action/audit/audit-action.xml:38` |
| `/api/auth.do` | GET/POST | 认证 帐号已经被禁用 106 网关已关闭口令认证服务。 107 帐号已经被吊销 108 账户已经被锁定 109 网关已经停止了Portal服务 110 密码错误,帐号已经被锁定. 111 您的账... | `pw, userName, password, encryption_status, client_encstatus, dataType, returnUrl, clientType, isreferer, codeIf, authMethod, tokenId, loginType, openid, mess...` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:63` |
| `/api/auth.do` | GET/POST | 认证 帐号已经被禁用 106 网关已关闭口令认证服务。 107 帐号已经被吊销 108 账户已经被锁定 109 网关已经停止了Portal服务 110 密码错误,帐号已经被锁定. 111 您的账... | `pw, userName, password, encryption_status, client_encstatus, dataType, returnUrl, clientType, isreferer, codeIf, authMethod, tokenId, loginType, openid, mess...` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:90` |
| `/api/authCheck.do` | GET/POST | 认证 帐号已经被禁用 106 网关已关闭口令认证服务。 107 帐号已经被吊销 108 账户已经被锁定 109 网关已经停止了Portal服务 110 密码错误,帐号已经被锁定. 111 您的账... | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:65` |
| `/api/authPolicy.do` | GET/POST | 获取认证策略接口 @return | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:88` |
| `/api/changeAppPwd.do` | GET/POST | - | `code, userName, password, key, dateType, appType` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:61` |
| `/api/changePwd.do` | GET/POST | 修改密码 100 密码修改成功 101 新的密码不能为空. 102 密码的长度为8~16个字符 103 密码强度不复合要求 104 修改失败，请检查Ldap连接是否正常 105 密码修改失败，第... | `dateType, password` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:74` |
| `/api/checkAuth.do` | GET/POST | 100.合法令牌 101.tokenId不能为空 102.令牌非法 | `dateType, tokenId` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:89` |
| `/api/checkRemoteToken.do` | GET/POST | - | `tokenId` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:6` |
| `/api/checkServerStatus.do` | GET/POST | 增加检测服务状态接口 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:9` |
| `/api/checkToken.do` | GET/POST | 根据token是否合法 | `tokenId` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:5` |
| `/api/clientState.do` | GET/POST | 客户端探针状态 | `tokenId, dataType, accountName, detail, startTime, endTime, visitTime, responseTime` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:170` |
| `/api/clientUpdate.do` | GET/POST | 客户端升级文件 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:48` |
| `/api/deviceBind.do` | GET/POST | 100 申请使用权限成功，等待审核! 101 终端不存在! 102 您的终端正在审核中，请审核后申请! 103 您的终端已经被吊销! 104 您的终端已经被禁用! 105 已申请过使用权限! 1... | `dataType` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:80` |
| `/api/deviceCheck.do` | GET/POST | 100 注册成功，并且自动审核成功!(终端检查通过!) 101 注册失败,该设备已经存在!! 102 您的终端正在审核中，请审核后登录! 103 您的终端已经被吊销，请联系管理员! 104 您的... | `dataType` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:78` |
| `/api/deviceReg.do` | GET/POST | 接口注册 100  注册成功. 101 注册失败,该设备已经存在 120 权限验证失败，您无权限使用该功能 102 注册成功，并且自动审核成功 | `dataType` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:79` |
| `/api/editAccountMap.do` | GET/POST | 编辑账户配置 | `appId, type, userName, password, options` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:128` |
| `/api/editFormAppAccount.do` | GET/POST | 帐户配置 | `appId, aacId, type, loginHTML, cmdHTML, userName, password` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:120` |
| `/api/editHTMLAppAcount.do` | GET/POST | 帐户配置 | `loginHTML, appId, type, %USER%, %PWD%` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:122` |
| `/api/editUserInfo.do` | GET/POST | 用户注册 修改失败,TokenId不合法 修改失败,非法操作 真实姓名不能为空 用户名不能为空 密码不能为空 密码的长度为8~16个字符 用户名重复 邮件重复 手机号重复 身份证号重复 | `tokenId, attr, appShowIp, accountId, status, gender, realName, email, mobile, telephone, birth, position, idCard, workID, workunit, deptId, userName, passwor...` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:25` |
| `/api/getAppAccountInfo.do` | GET/POST | 获得Ntml和Basic的帐户信息 | `appId, type` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:118` |
| `/api/getAppAccountList.do` | GET/POST | 获得账户应用列表 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:131` |
| `/api/getAppInfo.do` | GET/POST | 获得CS应用信息 | `id, type` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:94` |
| `/api/getAppList.do` | GET/POST | 获得应用列表 | `dateType, tokenId, subAccount` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:112` |
| `/api/getAppLists.do` | GET/POST | App应用 | `dateType, tokenId, subAccount` | `TrustMore/src/cn/com/tsg/admin/action/resource/resource-action.xml:49` |
| `/api/getAppLoginAccess.do` | GET/POST | 获得应用列表 | `appId, charSet` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:114` |
| `/api/getAppPolicy.do` | GET/POST | 获得应用策略 | `userId, userType, serviceId` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:96` |
| `/api/getBsAppList.do` | GET/POST | 获得应用列表 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:133` |
| `/api/getClearType.do` | GET/POST | 获取角色策略ClearType | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:67` |
| `/api/getCsAccountConfig.do` | GET/POST | 获得CS配置信息 | `appId` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:126` |
| `/api/getDNSForUser.do` | GET/POST | 获得用户dns策略 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:141` |
| `/api/getDeptPath.do` | GET/POST | 获得路径 | `id` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:30` |
| `/api/getDeptTree.do` | GET/POST | 获得部门树结构 | `id, deptId` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:29` |
| `/api/getICMPPolicy.do` | GET/POST | 获得ICMP策略 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:143` |
| `/api/getServerTime.do` | GET/POST | 获得服务器时间 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:10` |
| `/api/getWebFilterTactics.do` | GET/POST | 获得数据库过滤策略 | `name, targetIp, targetPort` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:137` |
| `/api/hb.do` | GET/POST | 心跳 | `tokenId, dataType` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:59` |
| `/api/htmlAppLogin.do` | GET/POST | 帐户配置 | `appId, type, am_id` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:124` |
| `/api/logout.do` | GET/POST | 用户注销 100 注销成功 120 非法操作. | `keyout, dateType, from_tsg_proxy, timeout, redirect, returnUrl, tokenId` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:66` |
| `/api/ncApp.do` | GET/POST | 角色策略下发 | `dateType, tokenId` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:149` |
| `/api/openApp.do` | GET/POST | 打开应用 | `id, type` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:57` |
| `/api/org.do` | GET/POST | - | `tokenId, dataType, orgId` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:107` |
| `/api/orgList.do` | GET/POST | 组织结构列表 | `page, size` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:101` |
| `/api/orgSync.do` | GET/POST | 部门同步 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:103` |
| `/api/portlet.do` | GET/POST | 迷你Portal | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:92` |
| `/api/resetPassWord.do` | GET/POST | 修改密码 100 密码修改成功 101 新的密码不能为空. 102 密码的长度为6~16个字符. 103 密码强度不复合要求. 104 修改失败，请检查Ldap连接是否正常. 105 密码修改失... | `dateType, newpass, oldpass` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:75` |
| `/api/rolePolicy.do` | GET/POST | 角色策略下发 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:146` |
| `/api/rolePolicyAll.do` | GET/POST | 下发的策略 | `clientType` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:147` |
| `/api/secondFactorAuth.do` | GET/POST | 二次认证接口 | `dataType, returnUrl, secondFactorAuthType, secondFactorAuthToken` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:64` |
| `/api/sendPageMessage.do` | GET/POST | 跳转代理返回错误信息提示页 | `clientType, message` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:182` |
| `/api/sendSms.do` | GET/POST | 100 成功 101 用户手机号不存在 102 用户不存在 | `userName, dataType` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:104` |
| `/api/sendSyslog.do` | GET/POST | 发送系统日志到Syslog服务器 | `targetIp, targetPort, action, name` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:138` |
| `/api/sendUserLog.do` | GET/POST | 角色策略配置不符合要求  记录审计日志 | `errorLog` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:173` |
| `/api/setAppSort.do` | GET/POST | 调用应用排序 | `dataType, type, json` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:179` |
| `/api/spaAuth.do` | GET/POST | spa认证 | `data, AuthIP` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:4` |
| `/api/testPortal.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:105` |
| `/api/updateSessionExt.do` | GET/POST | 把三方式系统更新session生命周期 | `tokenId, type, dataType` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:8` |
| `/api/userInfo.do` | GET/POST | 获得用户信息 | `key, apiId, dataType, userName` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:73` |
| `/api/userInfo.do` | GET/POST | 获得用户信息 | `key, apiId, dataType, userName` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:91` |
| `/api/userList.do` | GET/POST | 用户列表 | `page, size` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:100` |
| `/api/userSync.do` | GET/POST | 用户同步 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:102` |

### 5.2 文件/云盘接口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/api/share/checkPwd.do` | GET/POST | - | `id, password` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:16` |
| `/api/share/index.do` | GET/POST | - | `id, pid` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:15` |
| `/api/vpc/apiDemo.do` | GET/POST | ftp | `-` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:7` |
| `/api/vpc/cleanRecycleBin.do` | GET/POST | 清空会回收站 | `tokenId, dataType, appId` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:22` |
| `/api/vpc/delete.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:30` |
| `/api/vpc/deleteShareUser.do` | GET/POST | 删除分享用户 | `tokenId, dataType, fileId, appId, accountId` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:44` |
| `/api/vpc/downSF.do` | GET/POST | 下载 | `-` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:39` |
| `/api/vpc/download.do` | GET/POST | 下载文件 | `tokenId, dataType, share, appId, check, shareId, fileId` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:9` |
| `/api/vpc/filterUser.do` | GET/POST | - | `tokenId, dataType, realName` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:13` |
| `/api/vpc/getAppList.do` | GET/POST | 获得服务列表 | `dataType, tokenId` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:10` |
| `/api/vpc/getInfoByName.do` | GET/POST | 根据文件名获得信息 | `dataType, tokenId, parentId, appId, name` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:12` |
| `/api/vpc/getShareUserList.do` | GET/POST | 获得内部分享的所有用户 | `tokenId, dataType, fileId, appId` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:43` |
| `/api/vpc/getUploadCache.do` | POST | 获得上传缓存信息 | `tokenId, dataType, length, appId, md5` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:42` |
| `/api/vpc/list.do` | GET/POST | 显示文件列表 | `tokenId, dataType, page, pageSize, extType, share, name, folder, sub, recycle, shareId, appId, parentId` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:31` |
| `/api/vpc/mkdir.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:29` |
| `/api/vpc/popedom.do` | GET/POST | 检查策略 | `appId, action, path` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:34` |
| `/api/vpc/property.do` | GET/POST | 文件属性 | `tokenId, dataType, fileId, shareId, share` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:24` |
| `/api/vpc/rename.do` | GET/POST | 修改文件名 | `tokenId, dataType, targetFolderId, appId, targetName, fileId, shareId` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:32` |
| `/api/vpc/restore.do` | GET/POST | 从回收站还原 | `tokenId, dataType, appId, fileId, shareId` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:40` |
| `/api/vpc/saveLog.do` | GET/POST | 保存操作日志 | `tokenId, dataType, object, action, result, detailInfo` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:20` |
| `/api/vpc/sendEmail.do` | GET/POST | - | `dataType, tokenId, email, title, content` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:38` |
| `/api/vpc/share.do` | GET/POST | 分享文件 | `tokenId, dataType, appId, endTime, popedom, maxDownloadTimes, shareId, userId, type, fileId` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:33` |
| `/api/vpc/shareLog.do` | GET/POST | public void | `dateType, tokenId, shareId, appId, page, pageSize` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:18` |
| `/api/vpc/shareUrl.do` | GET/POST | 获得分享URL | `tokenId, dataType, type, popedom, docPopedom, appId, maxDownloadTimes, endTime, key, fileId, shareId` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:36` |
| `/api/vpc/updateShareInfo.do` | GET/POST | 更新分享参数信息 | `dataType, tokenId, popedom, docPopedom, appId, shareId, maxDownloadTimes, endTime, key` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:28` |
| `/api/vpc/upload.do` | POST | 上传文件 | `tokenId, dataType, md5, fileId, length, createTime, appId, path, parentId, shareId` | `TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:8` |

### 5.3 证书接口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/GetEncryptionCert.action` | GET/POST | 获取加密证书不能拦截 | `certType` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:20` |
| `/ajaxMenuTrustyCertDownload.do` | GET/POST | 下载文件 | `type, systemType` | `TrustMore/src/cn/com/tsg/admin/action/certmanager/cert-action.xml:61` |
| `/ajaxTrustCertBase64.do` | GET/POST | 下载文件 | `type` | `TrustMore/src/cn/com/tsg/admin/action/certmanager/cert-action.xml:62` |
| `/api/authCert.do` | GET/POST | 证书认证 for agent 100 认证成功 101 参数不合法 102 证书验证失败 103 证书已过期 104 证书已被吊销 | `type, cert, clientType, SourceAgent, User-Agent` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:76` |
| `/api/cert_auth` | GET/POST | web.xml 中单独映射到 CheckTwoWayUserFilter 的证书认证入口；未出现在 action XML 中。 | `双向证书认证上下文，具体参数由过滤器/URL handler 处理` | `TrustMore/WebContent/WEB-INF/web.xml:125` |
| `/api/verificationCert.do` | GET/POST | 对外提供校验证书有效性接口 不拦截 | `cert, verifylevel` | `TrustMore/src/cn/com/tsg/admin/action/auth/auth-action.xml:49` |
| `/api/verificationCertSn.do` | GET/POST | 验证该证书颁发者下证书是否被吊销 @return | `sn, issuer, ocsp` | `TrustMore/src/cn/com/tsg/admin/action/auth/auth-action.xml:50` |
| `/api/verificationCertTrust.do` | GET/POST | 增加校验外部证书方法 使用用户证书链的配置对证书进行进行校验 | `cert` | `TrustMore/src/cn/com/tsg/admin/action/auth/auth-action.xml:51` |
| `/authen/authCertService.action` | GET/POST | ukey证书认证 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:161` |
| `/certError.do` | GET/POST | - | `code, message` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:11` |
| `/certLogin.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:82` |
| `/userCertLogin.do` | GET/POST | 证书登录 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:17` |
| `/userportal/applyUserCert.action` | GET/POST | 申请用户证书 | `accountId, CertRequest, CertPass, certType` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:95` |
| `/userportal/certLogin.action` | GET/POST | 证书+口令认证接口 | `userName, password, returnUrl, host, clientType, randomStr, userCert, randomSign, client_encstatus` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:22` |
| `/userportal/certLoginGS.action` | GET/POST | 四川工商证书认证接口 | `randomStr, userCert, randomSign, keySid, bankType, bizType` | `TrustMore/src/cn/com/tsg/plugin/sichuanGS/sichuanGS-portal.xml:6` |
| `/userportal/checkCertOutOfDate.action` | GET/POST | 用户证书到期时间检查接口 | `userCert` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:149` |
| `/userportal/queryDeviceTrustyCert.action` | GET/POST | 下发设备证书链和用户证书链接口 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:152` |
| `/userverify/downloadAccountCert.action` | GET/POST | 下载用户证书 | `accountId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:99` |
| `/userverify/getUserCertStatus.action` | GET/POST | 获取用户证书审核状态 | `accountId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:97` |
| `/userverify/queryCertChainForbid.action` | GET/POST | 获取第三方证书链是否可导 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:158` |
| `/userverify/revokeAccountCert.action` | GET/POST | 吊销用户证书 | `accountId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:101` |

### 5.4 标准门户接口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/userportal/CCBUnitedGateway.action` | GET/POST | 四川工商建行国密规则查询接口 | `-` | `TrustMore/src/cn/com/tsg/plugin/sichuanGS/sichuanGS-portal.xml:9` |
| `/userportal/authPolicy.action` | GET/POST | 查询认证策略 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:105` |
| `/userportal/checkAccountInfo.action` | GET/POST | 校验用户信息是否重复 | `accountId, data, type` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:75` |
| `/userportal/checkAuthTokenId.action` | GET/POST | - | `tokenId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:146` |
| `/userportal/getAccountPolicy.action` | GET/POST | 获取用户策略 | `accountId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:14` |
| `/userportal/getNewVersion.action` | GET/POST | 获取升级版本号信息 | `type` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:135` |
| `/userportal/getOrgByParent.action` | GET/POST | 获取当前组织结构子级和用户 | `deptId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:128` |
| `/userportal/getSearcherUser.action` | GET/POST | 根据姓名，或者邮箱检索用户 | `searcher` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:132` |
| `/userportal/getUserByDeptId.action` | GET/POST | 组织机构下所有用户 | `deptId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:130` |
| `/userportal/getVersion.action` | GET/POST | 获取服务端版本号 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:7` |
| `/userportal/loginType.action` | GET/POST | 查询认证方式 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:93` |
| `/userportal/logout.action` | GET/POST | 注销接口 | `keyout, dateType, from_tsg_proxy, timeout, tokenId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:28` |
| `/userportal/messLogin.action` | GET/POST | 短信认证接口 | `userName, password, returnUrl, clientType, message, host, User-Agent` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:24` |
| `/userportal/passLogin.action` | GET/POST | 口令认证接口 | `userName, clientType, password, returnUrl, host, User-Agent` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:11` |
| `/userportal/queryDeptByPID.action` | GET/POST | 获取当前组织结构子级 | `deptId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:141` |
| `/userportal/queryDeptList.action` | GET/POST | 查询组织机构list接口 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:33` |
| `/userportal/queryUserTrustyKeyStore.action` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:153` |
| `/userportal/register.action` | GET/POST | 用户注册 | `status, gender, realName, email, mobile, telephone, birth, position, idCard, workID, workunit, deptId, userName, password, User-Agent` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:65` |
| `/userportal/test.action` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:5` |
| `/userportal/toLogin.action` | GET/POST | 需要登录 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:9` |
| `/userportal/uploadLogsInfo.action` | POST | - | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:155` |
| `/userportal/usbKLogin.action` | GET/POST | UsbKey认证接口 | `userName, returnUrl, clientType, authMethod, tokenId, host, randomStr, userCert, randomSign, client_encstatus` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:26` |

### 5.5 用户验证/管理接口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/userverify/NAAcountConfig.action` | GET/POST | N/A方式 设置从账号 | `appId, accountId, type, userName, password, User-Agent` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:71` |
| `/userverify/addDept.action` | GET/POST | 添加组织机构 | `name, shotName, enName, enShotName, code, parentId, User-Agent` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:39` |
| `/userverify/addRole.action` | GET/POST | 添加角色信息 | `name, note, whiteList, blackList, startTime, expireTime, User-Agent` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:59` |
| `/userverify/bindDevice.action` | GET/POST | 申请绑定 终端 | `deviceType, accountId, cpuId, cpuModel, bios, mainBoard, memory, vgaCard, driveModel, hdSerialNo, hdCapacity, mac, ip, osName, osVersion, installTime, exp1, ...` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:87` |
| `/userverify/changePwd.action` | GET/POST | 用户修改密码 | `oldpassword, newpassword` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:63` |
| `/userverify/checkDevice.action` | GET/POST | 终端检查 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:103` |
| `/userverify/delDeptByIds.action` | GET/POST | 根据id删除组织机构 | `deptId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:41` |
| `/userverify/delRoleByIds.action` | GET/POST | 根据id删除角色信息 | `roleId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:61` |
| `/userverify/delUserByIds.action` | GET/POST | 根据id删除用户信息 | `accountId, userState` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:51` |
| `/userverify/deviceList.action` | GET/POST | 获取终端列表 | `deviceState, userName, accountId, User-Agent` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:81` |
| `/userverify/deviceReg.action` | GET/POST | 终端注册 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:79` |
| `/userverify/getAccountIocn.action` | GET/POST | 获取用户头像 | `accountId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:109` |
| `/userverify/getAllEmail.action` | GET/POST | 获取邮箱信息 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:119` |
| `/userverify/getAllStorageSource.action` | GET/POST | 获取所有存储源 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:121` |
| `/userverify/getAllStorageSourceAuth.action` | GET/POST | 获取所有存用户存储源权限 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:123` |
| `/userverify/getDeviceById.action` | GET/POST | 根据id获取终端信息 | `id` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:83` |
| `/userverify/getFtpFilePostion.action` | GET/POST | - | `ip, port, user, password, path, fileName` | `TrustMore/src/cn/com/tsg/transfer/action/transfer-action.xml:5` |
| `/userverify/getHBeatCycleTime.action` | GET/POST | 获取心跳时间周期 | `clientType` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:144` |
| `/userverify/getOnlineDevice.action` | GET/POST | 获取在线用户终端 | `accountId, excludeLocal, tokenId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:89` |
| `/userverify/getProxyInfo.action` | GET/POST | 获取引擎IP和端口 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:125` |
| `/userverify/ncapp.action` | GET/POST | 获取nc引擎策略 | `dateType, tokenId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:164` |
| `/userverify/queryDeptById.action` | GET/POST | 根据id查询组织机构接口 | `deptId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:35` |
| `/userverify/queryHaveNetdisk.action` | GET/POST | 获取用户网盘权限 | `accountId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:116` |
| `/userverify/queryLogicDelUser.action` | GET/POST | 查询逻辑删除用户list接口 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:45` |
| `/userverify/queryNetdiskInfo.action` | GET/POST | 获取网盘配额 | `accountId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:113` |
| `/userverify/queryRoleById.action` | GET/POST | 根据id查询角色信息 | `roleId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:55` |
| `/userverify/queryRoleList.action` | GET/POST | 查询角色list接口 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:53` |
| `/userverify/queryUserById.action` | GET/POST | 根据id查询用户信息 | `accountId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:47` |
| `/userverify/queryUserList.action` | GET/POST | 查询用户list接口 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:43` |
| `/userverify/setDeviceUser.action` | GET/POST | 绑定/解绑 终端 | `deviceId, accountId, state` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:85` |
| `/userverify/updateDeptById.action` | GET/POST | 根据id修改组织机构接口 | `deptId, name, shotName, enName, enShotName, code, parentId, User-Agent` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:37` |
| `/userverify/updateRoleById.action` | GET/POST | 根据id修改角色信息 | `roleId, name, note, whiteList, blackList, startTime, expireTime, User-Agent` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:57` |
| `/userverify/updateUserById.action` | GET/POST | 根据id修改用户信息 | `accountId, gender, realName, email, mobile, telephone, birth, position, idCard, workID, workunit, deptId, User-Agent` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:49` |
| `/userverify/uploadAccountIcon.action` | POST | 上传用户头像 | `accountId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:111` |
| `/userverify/uploadLogsInfo.action` | POST | - | `clientType, logsData` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:154` |
| `/userverify/uploadTransfer.action` | POST | - | `ip, port, user, password, path, fileName, position` | `TrustMore/src/cn/com/tsg/transfer/action/transfer-action.xml:4` |
| `/userverify/userAppList.action` | GET/POST | 授权给用户的应用列表 | `subAccount, accountId` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:69` |

### 5.6 SSO/协议接口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/accounts/*` | GET/POST | OpenID 账号发现/处理入口；由 OpenIdFilter 拦截。 | `路径用户名，如 /accounts/{userName}` | `TrustMore/WebContent/WEB-INF/web.xml:279` |
| `/oauth/provider.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:51` |
| `/oauth/resources.do` | GET/POST | - | `appcode, tokenId, oauthAddr` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:52` |
| `/openid/provider.do` | GET/POST | - | `tokenId, account, appcode, ipAddr, _action, SourceHost` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:50` |
| `/saml/getArtifactResult.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:55` |
| `/saml/provider.do` | GET/POST | - | `_action` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:54` |

### 5.7 消息认证接口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/Authen/MessageService.action` | GET/POST | 报文认证 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:17` |
| `/Authen/SocketClientDown.action` | GET/POST | websocket客户端下载 、证书认证 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:16` |
| `/Authen/getRandom.action` | GET/POST | 获取随机数接口 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:167` |
| `/MessageService` | GET/POST | 报文认证 | `-` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:18` |

### 5.8 服务中心接口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/servicecenter/accessToken.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:6` |
| `/servicecenter/callbackCode.do` | GET/POST | 向服务管理中心注册接口 | `-` | `TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:5` |
| `/servicecenter/login.do` | GET/POST | 跳转统一认证接口 | `-` | `TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:8` |
| `/servicecenter/loginret.do` | GET/POST | 服务中心跳转业务服务接口 | `token` | `TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:7` |
| `/servicecenter/logout.do` | GET/POST | 登出接口 | `token` | `TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:9` |
| `/servicecenter/requestServerCode.do` | GET/POST | 向服务管理中心认证接口 | `-` | `TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:4` |

### 5.9 数据同步接口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/dataSyn/adduser.action` | GET/POST | 用户注册 | `status, gender, realName, email, mobile, telephone, birth, position, idCard, workID, workunit, deptId, userName, password, User-Agent` | `TrustMore/src/cn/com/tsg/syndata/action/DataSynchronization.xml:4` |
| `/dataSyn/checkUser.action` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/syndata/action/DataSynchronization.xml:5` |
| `/dataSyn/delete.action` | GET/POST | 删除用户 | `userName, password` | `TrustMore/src/cn/com/tsg/syndata/action/DataSynchronization.xml:7` |
| `/dataSyn/updatePassword.action` | GET/POST | 更新用户 | `userName, oldpassword, newpassword` | `TrustMore/src/cn/com/tsg/syndata/action/DataSynchronization.xml:6` |

### 5.10 零信任联动接口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/untrust/terminal/control.do` | GET/POST | 异常终端访问控制接口 | `-` | `TrustMore/src/cn/com/tsg/zerotrust/action/zeroTrustControl.xml:8` |
| `/untrust/user/control.do` | GET/POST | 异常用户访问控制接口 | `trustParam` | `TrustMore/src/cn/com/tsg/zerotrust/action/zeroTrustControl.xml:6` |

### 5.11 验证码接口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/captcha/check.do` | GET/POST | - | `data` | `TrustMore/src/cn/com/tsg/plugin/action/plugin-action.xml:14` |
| `/captcha/init.do` | GET/POST | Captcha | `-` | `TrustMore/src/cn/com/tsg/plugin/action/plugin-action.xml:13` |

### 5.12 门户兼容入口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/mobile/user/mobile_index.do` | GET/POST | 显示Portal页面 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:43` |
| `/portal6.2/api/logout.do` | GET/POST | 用户注销 100 注销成功 120 非法操作. | `keyout, dateType, from_tsg_proxy, timeout, redirect, returnUrl, tokenId` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:71` |
| `/portal6.2/user/index.do` | GET/POST | 显示Portal6.2页面 | `expDays, isreged, user-agent, SourceHost` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:40` |

### 5.13 登录/兼容入口

| 路径 | 方法 | 功能说明 | 参数摘要 | 来源 |
|---|---|---|---|---|
| `/APIDemoServlet` | GET/POST | API Demo Servlet，web.xml 直接 servlet-mapping。 | `method 等演示参数` | `TrustMore/WebContent/WEB-INF/web.xml:18` |
| `/activeXDownload.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/admin/action/certmanager/cert-action.xml:63` |
| `/clientLogin.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:16` |
| `/code.action` | GET/POST | 生成验证码 | `typeName` | `TrustMore/src/cn/com/tsg/admin/action/customPage/chosePage.xml:6` |
| `/code.do` | GET/POST | catch (Exception e) { e.printStackTrace(); }  } | `c, l, t` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:15` |
| `/dag_ws/v2/api/face/authreq.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:69` |
| `/deleteUserReg.do` | GET/POST | 删除注册用户 | `id` | `TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:67` |
| `/error.do` | GET/POST | - | `code, message` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:12` |
| `/initUserRegOk.do` | GET/POST | 用户注册初始化 | `accountId, code, status` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:19` |
| `/isCodeOk.action` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/admin/action/customPage/chosePage.xml:7` |
| `/linkReg.do` | GET/POST | 跳转用户注册页面 | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:18` |
| `/login.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:14` |
| `/mobile_updateUserInfo.do` | GET/POST | 修改个人信息 | `field, value` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:22` |
| `/proxyDebug.do` | GET/POST | - | `-` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:13` |
| `/userLogin.action` | GET/POST | 登录页选择 | `oauth, type, redirect_uri, SourceHost` | `TrustMore/src/cn/com/tsg/admin/action/customPage/chosePage.xml:5` |
| `/userRegOk.do` | GET/POST | 用户注册 修改失败,TokenId不合法 修改失败,非法操作 真实姓名不能为空 用户名不能为空 密码不能为空 密码的长度为8~16个字符 用户名重复 邮件重复 手机号重复 身份证号重复 | `tokenId, attr, appShowIp, accountId, status, gender, realName, email, mobile, telephone, birth, position, idCard, workID, workunit, deptId, userName, passwor...` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:20` |
| `/userUpdateUserInfo.do` | GET/POST | 修改个人信息 | `gender, email, mobile, telephone, birth, position, idCard, workID, realName, workunit, deptId` | `TrustMore/src/cn/com/tsg/user/action/user-action.xml:21` |

## 6. WebService / SOAP 接口

### `com.cmcc.mss.sb_mdm_hr_importpersoninfosrv.SBMDMHRImportPersonInfoSrvImpl`

- 源码：`TrustMore/src/com/cmcc/mss/sb_mdm_hr_importpersoninfosrv/SBMDMHRImportPersonInfoSrvImpl.java`

| 方法 | 参数 | 行 |
|---|---|---:|
| `process` | `-` | 38 |

### `com.cmcc.mss.sb_mdm_hr_importpersoninfosrv.SBMDMHRImportPersonInfoSrv`

- 源码：`TrustMore/src/com/cmcc/mss/sb_mdm_hr_importpersoninfosrv/SBMDMHRImportPersonInfoSrv.java`

| 方法 | 参数 | 行 |
|---|---|---:|
| `process` | `-` | 22 |

### `com.cmcc.mss.sb_mdm_hr_inquirymdmempolyeeinfosrv.SBMDMHRInquiryMDMEmpolyeeInfoSrv`

- 源码：`TrustMore/src/com/cmcc/mss/sb_mdm_hr_inquirymdmempolyeeinfosrv/SBMDMHRInquiryMDMEmpolyeeInfoSrv.java`

| 方法 | 参数 | 行 |
|---|---|---:|
| `process` | `-` | 22 |

### `com.cmcc.mss.sb_mdm_hr_inquirymdmempolyeeinfosrv.SBMDMHRInquiryMDMEmpolyeeInfoSrvImpl`

- 源码：`TrustMore/src/com/cmcc/mss/sb_mdm_hr_inquirymdmempolyeeinfosrv/SBMDMHRInquiryMDMEmpolyeeInfoSrvImpl.java`

| 方法 | 参数 | 行 |
|---|---|---:|
| `process` | `-` | 32 |

### `cn.com.tsg.servlet.IWebServiceApi`

- 源码：`TrustMore/src/cn/com/tsg/servlet/IWebServiceApi.java`
- 发布地址：`webservice_path` 配置项；默认 `http://localhost:8800/webserviceapi`

| 方法 | 参数 | 行 |
|---|---|---:|
| `auth` | `userName, password` | 8 |
| `sendMessages` | `phoneNum, message` | 9 |
| `getUrl` | `-` | 10 |

### `cn.com.tsg.servlet.WebServiceApi`

- 源码：`TrustMore/src/cn/com/tsg/servlet/WebServiceApi.java`
- 发布地址：`webservice_path` 配置项；默认 `http://localhost:8800/webserviceapi`

| 方法 | 参数 | 行 |
|---|---|---:|
| `auth` | `-` | 37 |
| `sendMessages` | `-` | 84 |
| `getUrl` | `-` | 112 |

### `cn.coremail.apiws.API`

- 源码：`TrustMore/src/cn/coremail/apiws/API.java`

| 方法 | 参数 | 行 |
|---|---|---:|
| `getVersionInfo` | `-` | 23 |
| `submit` | `cmd, params` | 26 |
| `createUser` | `-` | 34 |
| `deleteUser` | `-` | 42 |
| `changeAttrs` | `-` | 47 |
| `getAttrs` | `-` | 53 |
| `authenticate` | `-` | 59 |
| `userExist` | `-` | 65 |
| `userLogin` | `-` | 71 |
| `userLoginEx` | `-` | 76 |
| `userLogout` | `-` | 82 |
| `sesTimeOut` | `-` | 87 |
| `sesRefresh` | `-` | 92 |
| `getSesssionVar` | `ses_key` | 97 |
| `setSessionVar` | `ses_key, ses_var` | 103 |
| `addUnit` | `-` | 111 |
| `delUnit` | `-` | 118 |
| `getUnitAttrs` | `-` | 124 |
| `setUnitAttrs` | `-` | 131 |
| `addDomain25` | `-` | 139 |
| `delDomain25` | `-` | 144 |
| `domainExist` | `-` | 149 |
| `getDomainList` | `-` | 154 |
| `addDomainAlias` | `-` | 158 |
| `delDomainAlias` | `-` | 164 |
| `getDomainAlias` | `-` | 170 |
| `addOrg` | `-` | 176 |
| `alterOrg` | `-` | 182 |
| `getOrgInfo` | `-` | 188 |
| `addOrgDomain` | `-` | 194 |
| `delOrgDomain` | `-` | 200 |
| `addOrgCos` | `-` | 206 |
| `alterOrgCos` | `-` | 213 |
| `delOrgCos` | `-` | 220 |
| `getOrgListByDomain` | `-` | 226 |
| `getOrgList` | `-` | 231 |
| `getOrgCosUser` | `-` | 235 |
| `getOrgCosUserMax` | `-` | 241 |
| `addSmtpAlias` | `-` | 248 |
| `delSmtpAlias` | `-` | 254 |
| `getSmtpAlias` | `-` | 260 |
| `setAdminType` | `-` | 266 |
| `getAdminType` | `-` | 271 |
| `renameUser` | `-` | 276 |
| `moveUser` | `-` | 281 |
| `smtpTransport` | `mail_from, rcpt_to` | 287 |
| `getNewMailInfos` | `-` | 295 |

## 7. HTTP 接口详细定义

### 7.1 REST API

#### 1. `/api/accountCheck.do`

- **请求方法**：`GET/POST`
- **功能说明**：@return 1 成功；0失败。
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.accountCheck`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:97`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `passWord` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 2. `/api/adLogin.do`

- **请求方法**：`POST`
- **功能说明**：通过Active Directory进行域认证
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.adLogin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:28`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `account` | String | 是 | AD域用户名 |
| `password` | String | 是 | AD域密码 |
| `domain` | String | 否 | 域名 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `uid` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 3. `/api/appAccountConfig.do`

- **请求方法**：`POST`
- **功能说明**：获得应用列表
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.appAccountConfig`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:116`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `appId` | Integer | 是 | 应用ID |
| `appAccount` | String | 是 | 应用账号 |
| `appPassword` | String | 是 | 应用密码（加密） |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 4. `/api/appAuth.do`

- **请求方法**：`GET/POST`
- **功能说明**：应用帐号认证
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.appAuth`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:87`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `returnUrl` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appCode` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 5. `/api/audit/queryLog.do`

- **请求方法**：`POST`
- **功能说明**：获得日志列表
- **处理类/方法**：`cn.com.tsg.admin.action.audit.LogInfoAction.ajaxList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/audit/audit-action.xml:38`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌（需管理员权限） |
| `startTime` | String | 否 | 开始时间 |
| `endTime` | String | 否 | 结束时间 |
| `account` | String | 否 | 用户名（筛选） |
| `action` | String | 否 | 操作类型（筛选） |
| `pageNo` | Integer | 否 | 页码，默认1 |
| `pageSize` | Integer | 否 | 每页数量，默认20 |
| `TableFrom` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tablename` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `page` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `rp` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `d4311` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `d4312` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `LevelStr` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `Username` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `object` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `subjectTypeStr` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `address` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `Action` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `ObjectType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `Result` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 6. `/api/auth.do`

- **请求方法**：`POST`
- **功能说明**：认证 帐号已经被禁用 106 网关已关闭口令认证服务。 107 帐号已经被吊销 108 账户已经被锁定 109 网关已经停止了Portal服务 110 密码错误,帐号已经被锁定. 111 您的账...
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.auth`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:63`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `pw` | String | 是 | 密码（已加密） |
| `account` | String | 是 | 用户名 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `cert` | String | 否 | 客户端证书（Base64编码） |
| `type` | Integer | 否 | 认证类型 |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `encryption_status` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `client_encstatus` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `returnUrl` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `clientType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `isreferer` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `codeIf` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `authMethod` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `loginType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `openid` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `mess` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 7. `/api/auth.do`

- **请求方法**：`POST`
- **功能说明**：认证 帐号已经被禁用 106 网关已关闭口令认证服务。 107 帐号已经被吊销 108 账户已经被锁定 109 网关已经停止了Portal服务 110 密码错误,帐号已经被锁定. 111 您的账...
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.auth`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:90`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `pw` | String | 是 | 密码（已加密） |
| `account` | String | 是 | 用户名 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `cert` | String | 否 | 客户端证书（Base64编码） |
| `type` | Integer | 否 | 认证类型 |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `encryption_status` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `client_encstatus` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `returnUrl` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `clientType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `isreferer` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `codeIf` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `authMethod` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `loginType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `openid` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `mess` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 8. `/api/authCheck.do`

- **请求方法**：`POST`
- **功能说明**：认证 帐号已经被禁用 106 网关已关闭口令认证服务。 107 帐号已经被吊销 108 账户已经被锁定 109 网关已经停止了Portal服务 110 密码错误,帐号已经被锁定. 111 您的账...
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.authCheck`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:65`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `account` | String | 是 | 用户名 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 9. `/api/authPolicy.do`

- **请求方法**：`GET/POST`
- **功能说明**：获取认证策略接口 @return
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.authPolicy`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:88`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 10. `/api/changeAppPwd.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.changeAppPwd`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:61`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `code` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `key` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 11. `/api/changePwd.do`

- **请求方法**：`POST`
- **功能说明**：修改密码 100 密码修改成功 101 新的密码不能为空. 102 密码的长度为8~16个字符 103 密码强度不复合要求 104 修改失败，请检查Ldap连接是否正常 105 密码修改失败，第...
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.changePwd`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:74`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `oldPassword` | String | 是 | 原密码（加密） |
| `newPassword` | String | 是 | 新密码（加密） |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 12. `/api/checkAuth.do`

- **请求方法**：`GET/POST`
- **功能说明**：100.合法令牌 101.tokenId不能为空 102.令牌非法
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.checkAuth`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:89`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 13. `/api/checkRemoteToken.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.checkRemoteToken`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:6`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 14. `/api/checkServerStatus.do`

- **请求方法**：`POST`
- **功能说明**：增加检测服务状态接口
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.checkServerStatus`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:9`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 15. `/api/checkToken.do`

- **请求方法**：`GET/POST`
- **功能说明**：根据token是否合法
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.checkToken`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:5`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 16. `/api/clientState.do`

- **请求方法**：`GET/POST`
- **功能说明**：客户端探针状态
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.clientState`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:170`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `accountName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `detail` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `startTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `endTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `visitTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `responseTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 17. `/api/clientUpdate.do`

- **请求方法**：`POST`
- **功能说明**：客户端升级文件
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.clientUpdate`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:48`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `version` | String | 是 | 当前客户端版本 |
| `clientType` | String | 是 | 客户端类型：windows/linux/macos |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 18. `/api/deviceBind.do`

- **请求方法**：`POST`
- **功能说明**：100 申请使用权限成功，等待审核! 101 终端不存在! 102 您的终端正在审核中，请审核后申请! 103 您的终端已经被吊销! 104 您的终端已经被禁用! 105 已申请过使用权限! 1...
- **处理类/方法**：`cn.com.tsg.user.action.DeviceAction.bindApi`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:80`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `deviceId` | String | 是 | 设备ID |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 19. `/api/deviceCheck.do`

- **请求方法**：`POST`
- **功能说明**：100 注册成功，并且自动审核成功!(终端检查通过!) 101 注册失败,该设备已经存在!! 102 您的终端正在审核中，请审核后登录! 103 您的终端已经被吊销，请联系管理员! 104 您的...
- **处理类/方法**：`cn.com.tsg.user.action.DeviceAction.checkApi`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:78`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceId` | String | 是 | 设备ID |
| `tokenId` | String | 否 | 会话令牌 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 20. `/api/deviceReg.do`

- **请求方法**：`POST`
- **功能说明**：接口注册 100  注册成功. 101 注册失败,该设备已经存在 120 权限验证失败，您无权限使用该功能 102 注册成功，并且自动审核成功
- **处理类/方法**：`cn.com.tsg.user.action.DeviceAction.regApi`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:79`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `deviceId` | String | 是 | 设备ID |
| `deviceType` | String | 是 | 设备类型 |
| `deviceName` | String | 是 | 设备名称 |
| `mac` | String | 否 | MAC地址 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 21. `/api/editAccountMap.do`

- **请求方法**：`GET/POST`
- **功能说明**：编辑账户配置
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.editAccountMap`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:128`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `options` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 22. `/api/editFormAppAccount.do`

- **请求方法**：`GET/POST`
- **功能说明**：帐户配置
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.editFormAppAccount`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:120`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `aacId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `loginHTML` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `cmdHTML` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 23. `/api/editHTMLAppAcount.do`

- **请求方法**：`GET/POST`
- **功能说明**：帐户配置
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.editHTMLAppAcount`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:122`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `loginHTML` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `%USER%` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `%PWD%` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 24. `/api/editUserInfo.do`

- **请求方法**：`POST`
- **功能说明**：用户注册 修改失败,TokenId不合法 修改失败,非法操作 真实姓名不能为空 用户名不能为空 密码不能为空 密码的长度为8~16个字符 用户名重复 邮件重复 手机号重复 身份证号重复
- **处理类/方法**：`cn.com.tsg.user.action.UserAction.userRegOk`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:25`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `realName` | String | 否 | 真实姓名 |
| `email` | String | 否 | 电子邮箱 |
| `mobile` | String | 否 | 手机号码 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `attr` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appShowIp` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `status` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `gender` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `telephone` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `birth` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `position` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `idCard` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workID` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workunit` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `passwor` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 25. `/api/getAppAccountInfo.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得Ntml和Basic的帐户信息
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.getAppAccountInfo`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:118`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 26. `/api/getAppAccountList.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得账户应用列表
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.getAppAccountList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:131`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 27. `/api/getAppInfo.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得CS应用信息
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.getAppInfo`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:94`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 28. `/api/getAppList.do`

- **请求方法**：`POST`
- **功能说明**：获得应用列表
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.getAppList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:112`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `subAccount` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 29. `/api/getAppLists.do`

- **请求方法**：`GET/POST`
- **功能说明**：App应用
- **处理类/方法**：`cn.com.tsg.admin.action.resource.CsAppAction.getAppList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/resource/resource-action.xml:49`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `subAccount` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 30. `/api/getAppLoginAccess.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得应用列表
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.getAppLoginAccess`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:114`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `charSet` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 31. `/api/getAppPolicy.do`

- **请求方法**：`POST`
- **功能说明**：获得应用策略
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.getAppPolicy`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:96`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `appId` | Integer | 是 | 应用ID |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `userId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `serviceId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 32. `/api/getBsAppList.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得应用列表
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.getBsAppList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:133`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 33. `/api/getClearType.do`

- **请求方法**：`GET/POST`
- **功能说明**：获取角色策略ClearType
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.getClearType`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:67`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 34. `/api/getCsAccountConfig.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得CS配置信息
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.getCsAccountConfig`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:126`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 35. `/api/getDNSForUser.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得用户dns策略
- **处理类/方法**：`cn.com.tsg.user.action.DnsForUserAction.getDnsForUserPolicy`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:141`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 36. `/api/getDeptPath.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得路径
- **处理类/方法**：`cn.com.tsg.admin.action.account.DeptAction.getPath`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:30`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 37. `/api/getDeptTree.do`

- **请求方法**：`POST`
- **功能说明**：获得部门树结构
- **处理类/方法**：`cn.com.tsg.admin.action.account.DeptAction.getDeptTree`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:29`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `id` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 38. `/api/getICMPPolicy.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得ICMP策略
- **处理类/方法**：`cn.com.tsg.user.action.ICMPAction.getICMPPolicy`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:143`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 39. `/api/getServerTime.do`

- **请求方法**：`POST`
- **功能说明**：获得服务器时间
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.getServerTime`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:10`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 40. `/api/getWebFilterTactics.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得数据库过滤策略
- **处理类/方法**：`cn.com.tsg.user.action.WebFilterAction.getWebFilterTactics`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:137`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `name` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `targetIp` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `targetPort` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 41. `/api/hb.do`

- **请求方法**：`POST`
- **功能说明**：心跳
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.hb`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:59`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 42. `/api/htmlAppLogin.do`

- **请求方法**：`POST`
- **功能说明**：帐户配置
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.htmlAppLogin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:124`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `appId` | Integer | 是 | 应用ID |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `am_id` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 43. `/api/logout.do`

- **请求方法**：`POST`
- **功能说明**：用户注销 100 注销成功 120 非法操作.
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.logout`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:66`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `account` | String | 是 | 用户名 |
| `p` | String | 否 | 端口号 |
| `keyout` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `from_tsg_proxy` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `timeout` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `redirect` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `returnUrl` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 44. `/api/ncApp.do`

- **请求方法**：`GET/POST`
- **功能说明**：角色策略下发
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.ncApp`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:149`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 45. `/api/openApp.do`

- **请求方法**：`POST`
- **功能说明**：打开应用
- **处理类/方法**：`cn.com.tsg.user.action.PortalAction.openApp`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:57`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `appId` | Integer | 是 | 应用ID |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `id` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 46. `/api/org.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.org`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:107`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `orgId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 47. `/api/orgList.do`

- **请求方法**：`POST`
- **功能说明**：组织结构列表
- **处理类/方法**：`cn.com.tsg.user.action.DataSyncAction.orgList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:101`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `parentId` | Integer | 否 | 父节点ID，为空返回根节点 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `page` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `size` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 48. `/api/orgSync.do`

- **请求方法**：`GET/POST`
- **功能说明**：部门同步
- **处理类/方法**：`cn.com.tsg.user.action.DataSyncAction.orgSync`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:103`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 49. `/api/portlet.do`

- **请求方法**：`GET/POST`
- **功能说明**：迷你Portal
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.portlet`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:92`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 50. `/api/resetPassWord.do`

- **请求方法**：`POST`
- **功能说明**：修改密码 100 密码修改成功 101 新的密码不能为空. 102 密码的长度为6~16个字符. 103 密码强度不复合要求. 104 修改失败，请检查Ldap连接是否正常. 105 密码修改失...
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.resetPassWord`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:75`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 管理员会话令牌 |
| `account` | String | 是 | 目标用户名 |
| `newPassword` | String | 是 | 新密码（加密） |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `newpass` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `oldpass` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 51. `/api/rolePolicy.do`

- **请求方法**：`POST`
- **功能说明**：角色策略下发
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.rolePolicyDown`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:146`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `appId` | Integer | 是 | 应用ID |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 52. `/api/rolePolicyAll.do`

- **请求方法**：`GET/POST`
- **功能说明**：下发的策略
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.rolePolicy`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:147`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `clientType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 53. `/api/secondFactorAuth.do`

- **请求方法**：`POST`
- **功能说明**：二次认证接口
- **处理类/方法**：`cn.com.tsg.user.action.SecondFactorAuthAction.secondFactorAuth`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:64`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `account` | String | 是 | 用户名 |
| `factorType` | String | 是 | 认证因子类型：sms/email/token |
| `verifyCode` | String | 否 | 验证码（验证时必填） |
| `action` | String | 是 | 操作类型：send/verify |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `returnUrl` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `secondFactorAuthType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `secondFactorAuthToken` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 54. `/api/sendPageMessage.do`

- **请求方法**：`GET/POST`
- **功能说明**：跳转代理返回错误信息提示页
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.forwardErrorPage`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:182`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `clientType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `message` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 55. `/api/sendSms.do`

- **请求方法**：`POST`
- **功能说明**：100 成功 101 用户手机号不存在 102 用户不存在
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.sendSms`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:104`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `mobile` | String | 是 | 手机号码 |
| `content` | String | 是 | 短信内容 |
| `type` | String | 否 | 短信类型：verify/notify |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 56. `/api/sendSyslog.do`

- **请求方法**：`POST`
- **功能说明**：发送系统日志到Syslog服务器
- **处理类/方法**：`cn.com.tsg.user.action.WebFilterAction.sendSyslog`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:138`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `level` | String | 是 | 日志级别：DEBUG/INFO/WARN/ERROR |
| `message` | String | 是 | 日志消息 |
| `facility` | String | 否 | 设施类型 |
| `targetIp` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `targetPort` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `action` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `name` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 57. `/api/sendUserLog.do`

- **请求方法**：`POST`
- **功能说明**：角色策略配置不符合要求  记录审计日志
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.sendUserLog`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:173`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `action` | String | 是 | 操作类型 |
| `resource` | String | 是 | 操作资源 |
| `result` | String | 是 | 操作结果 |
| `detail` | String | 否 | 详细信息 |
| `errorLog` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 58. `/api/setAppSort.do`

- **请求方法**：`GET/POST`
- **功能说明**：调用应用排序
- **处理类/方法**：`cn.com.tsg.user.action.CustomAppAction.setAppSort`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:179`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `json` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 59. `/api/spaAuth.do`

- **请求方法**：`GET/POST`
- **功能说明**：spa认证
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.spaAuth`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:4`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `data` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `AuthIP` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 60. `/api/testPortal.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.testPortal`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:105`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 61. `/api/updateSessionExt.do`

- **请求方法**：`GET/POST`
- **功能说明**：把三方式系统更新session生命周期
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.updateSessionExt`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:8`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 62. `/api/userInfo.do`

- **请求方法**：`POST`
- **功能说明**：获得用户信息
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.userInfo`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:73`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌（Header或参数） |
| `apiId` | Integer | 否 | API ID，为0时使用tokenId认证 |
| `key` | String | 否 | API密钥 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 63. `/api/userInfo.do`

- **请求方法**：`POST`
- **功能说明**：获得用户信息
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.userInfo`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:91`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌（Header或参数） |
| `apiId` | Integer | 否 | API ID，为0时使用tokenId认证 |
| `key` | String | 否 | API密钥 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 64. `/api/userList.do`

- **请求方法**：`POST`
- **功能说明**：用户列表
- **处理类/方法**：`cn.com.tsg.user.action.DataSyncAction.userList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:100`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 是 | 会话令牌 |
| `deptId` | Integer | 否 | 部门ID（筛选） |
| `pageNo` | Integer | 否 | 页码，默认1 |
| `pageSize` | Integer | 否 | 每页数量，默认20 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `page` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `size` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 65. `/api/userSync.do`

- **请求方法**：`GET/POST`
- **功能说明**：用户同步
- **处理类/方法**：`cn.com.tsg.user.action.DataSyncAction.userSync`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:102`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

### 7.2 文件/云盘接口

#### 66. `/api/share/checkPwd.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FileShareAction.checkPwd`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:16`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 67. `/api/share/index.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FileShareAction.index`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:15`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `pid` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 68. `/api/vpc/apiDemo.do`

- **请求方法**：`GET/POST`
- **功能说明**：ftp
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.apiDemo`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:7`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 69. `/api/vpc/cleanRecycleBin.do`

- **请求方法**：`GET/POST`
- **功能说明**：清空会回收站
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.cleanRecycleBin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:22`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 70. `/api/vpc/delete.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.delete`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:30`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 71. `/api/vpc/deleteShareUser.do`

- **请求方法**：`GET/POST`
- **功能说明**：删除分享用户
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.deleteShareUser`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:44`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `fileId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 72. `/api/vpc/downSF.do`

- **请求方法**：`GET/POST`
- **功能说明**：下载
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.downloadShareFile`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:39`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 73. `/api/vpc/download.do`

- **请求方法**：`GET/POST`
- **功能说明**：下载文件
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.download`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:9`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `share` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `check` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shareId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `fileId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回文件流或下载响应。

#### 74. `/api/vpc/filterUser.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.filterUser`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:13`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `realName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 75. `/api/vpc/getAppList.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得服务列表
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.serviceList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:10`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 76. `/api/vpc/getInfoByName.do`

- **请求方法**：`GET/POST`
- **功能说明**：根据文件名获得信息
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.getInfoByName`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:12`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `parentId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `name` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 77. `/api/vpc/getShareUserList.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得内部分享的所有用户
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.shareUserList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:43`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `fileId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 78. `/api/vpc/getUploadCache.do`

- **请求方法**：`POST`
- **功能说明**：获得上传缓存信息
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.getUploadCache`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:42`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `length` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `md5` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回上传处理结果，格式以接口实现为准。

#### 79. `/api/vpc/list.do`

- **请求方法**：`GET/POST`
- **功能说明**：显示文件列表
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.list`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:31`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `page` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `pageSize` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `extType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `share` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `name` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `folder` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `sub` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `recycle` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shareId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `parentId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 80. `/api/vpc/mkdir.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.mkdir`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:29`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 81. `/api/vpc/popedom.do`

- **请求方法**：`GET/POST`
- **功能说明**：检查策略
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.popedom`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:34`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `action` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `path` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 82. `/api/vpc/property.do`

- **请求方法**：`GET/POST`
- **功能说明**：文件属性
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.property`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:24`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `fileId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shareId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `share` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 83. `/api/vpc/rename.do`

- **请求方法**：`GET/POST`
- **功能说明**：修改文件名
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.rename`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:32`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `targetFolderId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `targetName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `fileId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shareId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 84. `/api/vpc/restore.do`

- **请求方法**：`GET/POST`
- **功能说明**：从回收站还原
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.restore`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:40`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `fileId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shareId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 85. `/api/vpc/saveLog.do`

- **请求方法**：`GET/POST`
- **功能说明**：保存操作日志
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.saveLog`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:20`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `object` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `action` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `result` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `detailInfo` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 86. `/api/vpc/sendEmail.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.sendEmail`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:38`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `email` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `title` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `content` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 87. `/api/vpc/share.do`

- **请求方法**：`GET/POST`
- **功能说明**：分享文件
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.share`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:33`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `endTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `popedom` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `maxDownloadTimes` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shareId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `fileId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 88. `/api/vpc/shareLog.do`

- **请求方法**：`GET/POST`
- **功能说明**：public void
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FileShareAction.shareLog`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:18`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shareId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `page` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `pageSize` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 89. `/api/vpc/shareUrl.do`

- **请求方法**：`GET/POST`
- **功能说明**：获得分享URL
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.shareUrl`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:36`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `popedom` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `docPopedom` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `maxDownloadTimes` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `endTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `key` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `fileId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shareId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 90. `/api/vpc/updateShareInfo.do`

- **请求方法**：`GET/POST`
- **功能说明**：更新分享参数信息
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.updateShareInfo`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:28`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `popedom` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `docPopedom` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shareId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `maxDownloadTimes` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `endTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `key` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 91. `/api/vpc/upload.do`

- **请求方法**：`POST`
- **功能说明**：上传文件
- **处理类/方法**：`cn.com.tsg.plugin.ftp.action.FtpApiAction.upload`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/ftp/action/ftp-action.xml:8`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dataType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `md5` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `fileId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `length` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `createTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `path` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `parentId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shareId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回上传处理结果，格式以接口实现为准。

### 7.3 证书接口

#### 92. `/GetEncryptionCert.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取加密证书不能拦截
- **处理类/方法**：`cn.com.tsg.messageauth.action.GetEncryptionCertAction.getEncryptionCert`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:20`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `certType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 93. `/ajaxMenuTrustyCertDownload.do`

- **请求方法**：`GET`
- **功能说明**：下载文件
- **处理类/方法**：`cn.com.tsg.admin.action.certmanager.MenuTrustyCertAction.certdownload`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/certmanager/cert-action.xml:61`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `certId` | Integer | 是 | 证书ID |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `systemType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回文件流或下载响应。

#### 94. `/ajaxTrustCertBase64.do`

- **请求方法**：`POST`
- **功能说明**：下载文件
- **处理类/方法**：`cn.com.tsg.admin.action.certmanager.MenuTrustyCertAction.trustCertBase64`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/certmanager/cert-action.xml:62`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `certId` | Integer | 是 | 证书ID |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 95. `/api/authCert.do`

- **请求方法**：`POST`
- **功能说明**：证书认证 for agent 100 认证成功 101 参数不合法 102 证书验证失败 103 证书已过期 104 证书已被吊销
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.authCert`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:76`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `cert` | String | 是 | 客户端证书（Base64编码） |
| `type` | Integer | 否 | 证书类型 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `clientType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `SourceAgent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 96. `/api/cert_auth`

- **请求方法**：`GET/POST`
- **功能说明**：web.xml 中单独映射到 CheckTwoWayUserFilter 的证书认证入口；未出现在 action XML 中。
- **处理类/方法**：`cn.com.tsg.util.CheckTwoWayUserFilter / cn.com.tsg.urlhander.BuildUrlHander`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/WebContent/WEB-INF/web.xml:125`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `双向证书认证上下文，具体参数由过滤器/URL handler 处理` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 97. `/api/verificationCert.do`

- **请求方法**：`POST`
- **功能说明**：对外提供校验证书有效性接口 不拦截
- **处理类/方法**：`cn.com.tsg.admin.action.auth.CertVerificationAction.CertVerification`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/auth/auth-action.xml:49`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `cert` | String | 是 | 待验证的证书（Base64） |
| `checkCRL` | Boolean | 否 | 是否检查CRL，默认true |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `verifylevel` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 98. `/api/verificationCertSn.do`

- **请求方法**：`POST`
- **功能说明**：验证该证书颁发者下证书是否被吊销 @return
- **处理类/方法**：`cn.com.tsg.admin.action.auth.CertVerificationAction.CertVerificationBySn`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/auth/auth-action.xml:50`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `serialNumber` | String | 是 | 证书序列号 |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |
| `sn` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `issuer` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `ocsp` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 99. `/api/verificationCertTrust.do`

- **请求方法**：`POST`
- **功能说明**：增加校验外部证书方法 使用用户证书链的配置对证书进行进行校验
- **处理类/方法**：`cn.com.tsg.admin.action.auth.CertVerificationAction.CertVerificationTrust`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/auth/auth-action.xml:51`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `cert` | String | 是 | 证书（Base64） |
| `dataType` | String | 否 | 返回数据类型：xml/json，默认 xml |

**响应说明**

- 旧文档包含响应示例或响应参数说明；具体返回以接口实现为准。

#### 100. `/authen/authCertService.action`

- **请求方法**：`GET/POST`
- **功能说明**：ukey证书认证
- **处理类/方法**：`cn.com.tsg.standard.action.AccountCertInterface.authCert`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:161`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 101. `/certError.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.errorHandle`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:11`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `code` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `message` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 102. `/certLogin.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.user.action.UserAction.certLogin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:82`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 103. `/userCertLogin.do`

- **请求方法**：`GET/POST`
- **功能说明**：证书登录
- **处理类/方法**：`cn.com.tsg.user.action.UserAction.userCertLogin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:17`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 104. `/userportal/applyUserCert.action`

- **请求方法**：`GET/POST`
- **功能说明**：申请用户证书
- **处理类/方法**：`cn.com.tsg.standard.action.AccountCertInterface.createUserCert`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:95`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `CertRequest` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `CertPass` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `certType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 105. `/userportal/certLogin.action`

- **请求方法**：`GET/POST`
- **功能说明**：证书+口令认证接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.certificateLogin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:22`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `returnUrl` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `host` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `clientType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `randomStr` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userCert` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `randomSign` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `client_encstatus` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 106. `/userportal/certLoginGS.action`

- **请求方法**：`GET/POST`
- **功能说明**：四川工商证书认证接口
- **处理类/方法**：`cn.com.tsg.plugin.sichuanGS.GSAction.certLoginSC`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/sichuanGS/sichuanGS-portal.xml:6`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `randomStr` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userCert` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `randomSign` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `keySid` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `bankType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `bizType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 107. `/userportal/checkCertOutOfDate.action`

- **请求方法**：`GET/POST`
- **功能说明**：用户证书到期时间检查接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.checkCertOutOfDate`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:149`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `userCert` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 108. `/userportal/queryDeviceTrustyCert.action`

- **请求方法**：`GET/POST`
- **功能说明**：下发设备证书链和用户证书链接口
- **处理类/方法**：`cn.com.tsg.standard.action.CertificatesTrustyAction.provideDeviceTrustyCert`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:152`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 109. `/userverify/downloadAccountCert.action`

- **请求方法**：`GET/POST`
- **功能说明**：下载用户证书
- **处理类/方法**：`cn.com.tsg.standard.action.AccountCertInterface.downloadAccountCert`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:99`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回文件流或下载响应。

#### 110. `/userverify/getUserCertStatus.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取用户证书审核状态
- **处理类/方法**：`cn.com.tsg.standard.action.AccountCertInterface.getAccountCertStatus`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:97`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 111. `/userverify/queryCertChainForbid.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取第三方证书链是否可导
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.getCertChainForbid`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:158`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 112. `/userverify/revokeAccountCert.action`

- **请求方法**：`GET/POST`
- **功能说明**：吊销用户证书
- **处理类/方法**：`cn.com.tsg.standard.action.AccountCertInterface.revokeAccountCert`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:101`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

### 7.4 标准门户接口

#### 113. `/userportal/CCBUnitedGateway.action`

- **请求方法**：`GET/POST`
- **功能说明**：四川工商建行国密规则查询接口
- **处理类/方法**：`cn.com.tsg.plugin.sichuanGS.GSAction.CCBUnitedGateway`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/sichuanGS/sichuanGS-portal.xml:9`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 114. `/userportal/authPolicy.action`

- **请求方法**：`GET/POST`
- **功能说明**：查询认证策略
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.authPolicy`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:105`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 115. `/userportal/checkAccountInfo.action`

- **请求方法**：`GET/POST`
- **功能说明**：校验用户信息是否重复
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.checkAccountInfo`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:75`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `data` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 116. `/userportal/checkAuthTokenId.action`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.checkTokenIdToPortal`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:146`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 117. `/userportal/getAccountPolicy.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取用户策略
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.getAccountPolicy`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:14`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 118. `/userportal/getNewVersion.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取升级版本号信息
- **处理类/方法**：`cn.com.tsg.standard.action.SystemPortal.getNewVersion`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:135`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 119. `/userportal/getOrgByParent.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取当前组织结构子级和用户
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryDeptByParent`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:128`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 120. `/userportal/getSearcherUser.action`

- **请求方法**：`GET/POST`
- **功能说明**：根据姓名，或者邮箱检索用户
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryUserByUserdefinedFiled`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:132`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `searcher` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 121. `/userportal/getUserByDeptId.action`

- **请求方法**：`GET/POST`
- **功能说明**：组织机构下所有用户
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryUserChildByDeptId`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:130`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 122. `/userportal/getVersion.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取服务端版本号
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.getVersion`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:7`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 123. `/userportal/loginType.action`

- **请求方法**：`GET/POST`
- **功能说明**：查询认证方式
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.loginType`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:93`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 124. `/userportal/logout.action`

- **请求方法**：`GET/POST`
- **功能说明**：注销接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.logout`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:28`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `keyout` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `from_tsg_proxy` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `timeout` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 125. `/userportal/messLogin.action`

- **请求方法**：`GET/POST`
- **功能说明**：短信认证接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.messageLogin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:24`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `returnUrl` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `clientType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `message` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `host` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 126. `/userportal/passLogin.action`

- **请求方法**：`GET/POST`
- **功能说明**：口令认证接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.passwordLogin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:11`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `clientType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `returnUrl` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `host` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 127. `/userportal/queryDeptByPID.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取当前组织结构子级
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryDeptByPId`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:141`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 128. `/userportal/queryDeptList.action`

- **请求方法**：`GET/POST`
- **功能说明**：查询组织机构list接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryDeptList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:33`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 129. `/userportal/queryUserTrustyKeyStore.action`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.standard.action.CertificatesTrustyAction.provideUserTrustyKeyStore`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:153`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 130. `/userportal/register.action`

- **请求方法**：`GET/POST`
- **功能说明**：用户注册
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.register`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:65`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `status` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `gender` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `realName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `email` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `mobile` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `telephone` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `birth` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `position` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `idCard` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workID` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workunit` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 131. `/userportal/test.action`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.test`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:5`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 132. `/userportal/toLogin.action`

- **请求方法**：`GET/POST`
- **功能说明**：需要登录
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.toLogin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:9`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 133. `/userportal/uploadLogsInfo.action`

- **请求方法**：`POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.standard.action.CertificatesTrustyAction.uploadLogsForClientNew`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:155`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回上传处理结果，格式以接口实现为准。

#### 134. `/userportal/usbKLogin.action`

- **请求方法**：`GET/POST`
- **功能说明**：UsbKey认证接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.usbKeyLogin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:26`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `returnUrl` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `clientType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `authMethod` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `host` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `randomStr` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userCert` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `randomSign` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `client_encstatus` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

### 7.5 用户验证/管理接口

#### 135. `/userverify/NAAcountConfig.action`

- **请求方法**：`GET/POST`
- **功能说明**：N/A方式 设置从账号
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.NAAcountConfig`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:71`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `appId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 136. `/userverify/addDept.action`

- **请求方法**：`GET/POST`
- **功能说明**：添加组织机构
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.addDept`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:39`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `name` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shotName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `enName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `enShotName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `code` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `parentId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 137. `/userverify/addRole.action`

- **请求方法**：`GET/POST`
- **功能说明**：添加角色信息
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.addRole`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:59`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `name` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `note` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `whiteList` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `blackList` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `startTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `expireTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 138. `/userverify/bindDevice.action`

- **请求方法**：`GET/POST`
- **功能说明**：申请绑定 终端
- **处理类/方法**：`cn.com.tsg.standard.action.DevicePortal.bindDevice`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:87`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `cpuId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `cpuModel` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `bios` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `mainBoard` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `memory` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `vgaCard` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `driveModel` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `hdSerialNo` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `hdCapacity` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `mac` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `ip` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `osName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `osVersion` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `installTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `exp1` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 139. `/userverify/changePwd.action`

- **请求方法**：`GET/POST`
- **功能说明**：用户修改密码
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.changePwd`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:63`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `oldpassword` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `newpassword` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 140. `/userverify/checkDevice.action`

- **请求方法**：`GET/POST`
- **功能说明**：终端检查
- **处理类/方法**：`cn.com.tsg.standard.action.DevicePortal.checkDevice`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:103`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 141. `/userverify/delDeptByIds.action`

- **请求方法**：`GET/POST`
- **功能说明**：根据id删除组织机构
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.delDeptNode`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:41`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 142. `/userverify/delRoleByIds.action`

- **请求方法**：`GET/POST`
- **功能说明**：根据id删除角色信息
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.delRoleByIds`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:61`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `roleId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 143. `/userverify/delUserByIds.action`

- **请求方法**：`GET/POST`
- **功能说明**：根据id删除用户信息
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.delUserByIds`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:51`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userState` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 144. `/userverify/deviceList.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取终端列表
- **处理类/方法**：`cn.com.tsg.standard.action.DevicePortal.list`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:81`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceState` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 145. `/userverify/deviceReg.action`

- **请求方法**：`GET/POST`
- **功能说明**：终端注册
- **处理类/方法**：`cn.com.tsg.standard.action.DevicePortal.reg`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:79`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 146. `/userverify/getAccountIocn.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取用户头像
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.getAccountIcon`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:109`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 147. `/userverify/getAllEmail.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取邮箱信息
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.getAllEmail`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:119`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 148. `/userverify/getAllStorageSource.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取所有存储源
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.getAllStorageSource`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:121`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 149. `/userverify/getAllStorageSourceAuth.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取所有存用户存储源权限
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.getAllStorageSourceAuth`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:123`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 150. `/userverify/getDeviceById.action`

- **请求方法**：`GET/POST`
- **功能说明**：根据id获取终端信息
- **处理类/方法**：`cn.com.tsg.standard.action.DevicePortal.get`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:83`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 151. `/userverify/getFtpFilePostion.action`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.transfer.action.TransferAction.getFilePostion`
- **是否在 DispatcherFilter 加载**：否
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/transfer/action/transfer-action.xml:5`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `ip` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `port` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `user` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `path` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `fileName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 152. `/userverify/getHBeatCycleTime.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取心跳时间周期
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryHBCycleTime`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:144`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `clientType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 153. `/userverify/getOnlineDevice.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取在线用户终端
- **处理类/方法**：`cn.com.tsg.standard.action.DevicePortal.getOnlineDevice`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:89`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `excludeLocal` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 154. `/userverify/getProxyInfo.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取引擎IP和端口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.getProxyInfo`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:125`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 155. `/userverify/ncapp.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取nc引擎策略
- **处理类/方法**：`cn.com.tsg.standard.action.AppPortal.ncApp`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:164`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 156. `/userverify/queryDeptById.action`

- **请求方法**：`GET/POST`
- **功能说明**：根据id查询组织机构接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryDeptById`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:35`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 157. `/userverify/queryHaveNetdisk.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取用户网盘权限
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryHaveNetdisk`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:116`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 158. `/userverify/queryLogicDelUser.action`

- **请求方法**：`GET/POST`
- **功能说明**：查询逻辑删除用户list接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryLogicDelUserList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:45`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 159. `/userverify/queryNetdiskInfo.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取网盘配额
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryNetdiskUserSize`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:113`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 160. `/userverify/queryRoleById.action`

- **请求方法**：`GET/POST`
- **功能说明**：根据id查询角色信息
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryRoleById`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:55`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `roleId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 161. `/userverify/queryRoleList.action`

- **请求方法**：`GET/POST`
- **功能说明**：查询角色list接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryRoleList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:53`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 162. `/userverify/queryUserById.action`

- **请求方法**：`GET/POST`
- **功能说明**：根据id查询用户信息
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryUserById`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:47`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 163. `/userverify/queryUserList.action`

- **请求方法**：`GET/POST`
- **功能说明**：查询用户list接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.queryUserList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:43`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 164. `/userverify/setDeviceUser.action`

- **请求方法**：`GET/POST`
- **功能说明**：绑定/解绑 终端
- **处理类/方法**：`cn.com.tsg.standard.action.DevicePortal.setDeviceUser`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:85`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `state` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 165. `/userverify/updateDeptById.action`

- **请求方法**：`GET/POST`
- **功能说明**：根据id修改组织机构接口
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.updateDeptById`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:37`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `name` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `shotName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `enName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `enShotName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `code` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `parentId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 166. `/userverify/updateRoleById.action`

- **请求方法**：`GET/POST`
- **功能说明**：根据id修改角色信息
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.updateRoleById`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:57`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `roleId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `name` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `note` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `whiteList` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `blackList` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `startTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `expireTime` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 167. `/userverify/updateUserById.action`

- **请求方法**：`GET/POST`
- **功能说明**：根据id修改用户信息
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.updateUserById`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:49`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `gender` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `realName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `email` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `mobile` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `telephone` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `birth` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `position` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `idCard` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workID` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workunit` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 168. `/userverify/uploadAccountIcon.action`

- **请求方法**：`POST`
- **功能说明**：上传用户头像
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.uploadAccountIcon`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:111`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回上传处理结果，格式以接口实现为准。

#### 169. `/userverify/uploadLogsInfo.action`

- **请求方法**：`POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.standard.action.CertificatesTrustyAction.uploadLogsForClient`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:154`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `clientType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `logsData` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回上传处理结果，格式以接口实现为准。

#### 170. `/userverify/uploadTransfer.action`

- **请求方法**：`POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.transfer.action.TransferAction.upload`
- **是否在 DispatcherFilter 加载**：否
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/transfer/action/transfer-action.xml:4`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `ip` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `port` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `user` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `path` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `fileName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `position` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回上传处理结果，格式以接口实现为准。

#### 171. `/userverify/userAppList.action`

- **请求方法**：`GET/POST`
- **功能说明**：授权给用户的应用列表
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.userAppList`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:69`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `subAccount` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

### 7.6 SSO/协议接口

#### 172. `/accounts/*`

- **请求方法**：`GET/POST`
- **功能说明**：OpenID 账号发现/处理入口；由 OpenIdFilter 拦截。
- **处理类/方法**：`cn.com.tsg.openid.filter.OpenIdFilter`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/WebContent/WEB-INF/web.xml:279`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `路径用户名，如 /accounts/{userName}` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 173. `/oauth/provider.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.oauth.OauthAction.provider`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:51`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 174. `/oauth/resources.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.oauth.OauthAction.resources`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:52`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `appcode` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `oauthAddr` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 175. `/openid/provider.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.openid.action.ProviderAction.provider`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:50`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `account` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appcode` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `ipAddr` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `_action` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `SourceHost` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 176. `/saml/getArtifactResult.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.saml.action.ProviderAction.getArtifactResult`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:55`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 177. `/saml/provider.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.saml.action.ProviderAction.provider`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:54`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `_action` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

### 7.7 消息认证接口

#### 178. `/Authen/MessageService.action`

- **请求方法**：`GET/POST`
- **功能说明**：报文认证
- **处理类/方法**：`cn.com.tsg.messageauth.action.MsgAuthenAction.messageAuth`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:17`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 179. `/Authen/SocketClientDown.action`

- **请求方法**：`GET/POST`
- **功能说明**：websocket客户端下载 、证书认证
- **处理类/方法**：`cn.com.tsg.messageauth.action.WebSocketClientDownAction.webSocketClientDown`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:16`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 180. `/Authen/getRandom.action`

- **请求方法**：`GET/POST`
- **功能说明**：获取随机数接口
- **处理类/方法**：`cn.com.tsg.messageauth.action.GetRandomAction.getRandom`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:167`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 181. `/MessageService`

- **请求方法**：`GET/POST`
- **功能说明**：报文认证
- **处理类/方法**：`cn.com.tsg.messageauth.action.MsgAuthenAction.messageAuth`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:18`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

### 7.8 服务中心接口

#### 182. `/servicecenter/accessToken.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.admin.action.servicecenter.ServiceCenterAction.accessToken`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:6`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 183. `/servicecenter/callbackCode.do`

- **请求方法**：`GET/POST`
- **功能说明**：向服务管理中心注册接口
- **处理类/方法**：`cn.com.tsg.admin.action.servicecenter.ServiceCenterAction.callbackCode`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:5`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 184. `/servicecenter/login.do`

- **请求方法**：`GET/POST`
- **功能说明**：跳转统一认证接口
- **处理类/方法**：`cn.com.tsg.admin.action.servicecenter.OAuthAction.login`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:8`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 185. `/servicecenter/loginret.do`

- **请求方法**：`GET/POST`
- **功能说明**：服务中心跳转业务服务接口
- **处理类/方法**：`cn.com.tsg.admin.action.servicecenter.OAuthAction.loginSuccess`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:7`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `token` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 186. `/servicecenter/logout.do`

- **请求方法**：`GET/POST`
- **功能说明**：登出接口
- **处理类/方法**：`cn.com.tsg.admin.action.servicecenter.OAuthAction.logout`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:9`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `token` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 187. `/servicecenter/requestServerCode.do`

- **请求方法**：`GET/POST`
- **功能说明**：向服务管理中心认证接口
- **处理类/方法**：`cn.com.tsg.admin.action.servicecenter.ServiceCenterAction.requestServerCode`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/servicecenter/service-center-action.xml:4`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

### 7.9 数据同步接口

#### 188. `/dataSyn/adduser.action`

- **请求方法**：`GET/POST`
- **功能说明**：用户注册
- **处理类/方法**：`cn.com.tsg.syndata.action.DataSynchronization.adduser`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/syndata/action/DataSynchronization.xml:4`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `status` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `gender` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `realName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `email` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `mobile` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `telephone` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `birth` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `position` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `idCard` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workID` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workunit` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `User-Agent` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 189. `/dataSyn/checkUser.action`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.syndata.action.DataSynchronization.execute`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/syndata/action/DataSynchronization.xml:5`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 190. `/dataSyn/delete.action`

- **请求方法**：`GET/POST`
- **功能说明**：删除用户
- **处理类/方法**：`cn.com.tsg.syndata.action.DataSynchronization.delete`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/syndata/action/DataSynchronization.xml:7`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `password` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 191. `/dataSyn/updatePassword.action`

- **请求方法**：`GET/POST`
- **功能说明**：更新用户
- **处理类/方法**：`cn.com.tsg.syndata.action.DataSynchronization.updatePassword`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/syndata/action/DataSynchronization.xml:6`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `oldpassword` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `newpassword` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

### 7.10 零信任联动接口

#### 192. `/untrust/terminal/control.do`

- **请求方法**：`GET/POST`
- **功能说明**：异常终端访问控制接口
- **处理类/方法**：`cn.com.tsg.zerotrust.action.ZeroTrustControlAction.controlUnusualTerminal`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/zerotrust/action/zeroTrustControl.xml:8`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 193. `/untrust/user/control.do`

- **请求方法**：`GET/POST`
- **功能说明**：异常用户访问控制接口
- **处理类/方法**：`cn.com.tsg.zerotrust.action.ZeroTrustControlAction.controlUnusualUser`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/zerotrust/action/zeroTrustControl.xml:6`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `trustParam` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

### 7.11 验证码接口

#### 194. `/captcha/check.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.plugin.captcha.action.CaptchaAction.check`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/action/plugin-action.xml:14`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `data` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 195. `/captcha/init.do`

- **请求方法**：`GET/POST`
- **功能说明**：Captcha
- **处理类/方法**：`cn.com.tsg.plugin.captcha.action.CaptchaAction.init`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/plugin/action/plugin-action.xml:13`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

### 7.12 门户兼容入口

#### 196. `/mobile/user/mobile_index.do`

- **请求方法**：`GET/POST`
- **功能说明**：显示Portal页面
- **处理类/方法**：`cn.com.tsg.user.action.PortalAction.mobile_index`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:43`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 197. `/portal6.2/api/logout.do`

- **请求方法**：`GET/POST`
- **功能说明**：用户注销 100 注销成功 120 非法操作.
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.logout`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:71`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `keyout` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `dateType` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `from_tsg_proxy` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `timeout` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `redirect` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `returnUrl` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 198. `/portal6.2/user/index.do`

- **请求方法**：`GET/POST`
- **功能说明**：显示Portal6.2页面
- **处理类/方法**：`cn.com.tsg.user.action.PortalAction.index62`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:40`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `expDays` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `isreged` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `user-agent` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `SourceHost` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

### 7.13 登录/兼容入口

#### 199. `/APIDemoServlet`

- **请求方法**：`GET/POST`
- **功能说明**：API Demo Servlet，web.xml 直接 servlet-mapping。
- **处理类/方法**：`com.tebie.APIDemo.servlet.APIDemoServlet`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/WebContent/WEB-INF/web.xml:18`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `method 等演示参数` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 200. `/activeXDownload.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.admin.action.certmanager.MenuTrustyCertAction.activeXDownload`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/certmanager/cert-action.xml:63`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回文件流或下载响应。

#### 201. `/clientLogin.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.user.action.UserAction.clientLogin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:16`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 202. `/code.action`

- **请求方法**：`GET/POST`
- **功能说明**：生成验证码
- **处理类/方法**：`cn.com.tsg.admin.action.customPage.CustomAction.code`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/customPage/chosePage.xml:6`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `typeName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 203. `/code.do`

- **请求方法**：`GET/POST`
- **功能说明**：catch (Exception e) { e.printStackTrace(); }  }
- **处理类/方法**：`cn.com.tsg.user.action.ValidateCodeAction.execute`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:15`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `c` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `l` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `t` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 204. `/dag_ws/v2/api/face/authreq.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.authreq`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:69`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 205. `/deleteUserReg.do`

- **请求方法**：`GET/POST`
- **功能说明**：删除注册用户
- **处理类/方法**：`cn.com.tsg.standard.action.UserPortal.deleteUserReg`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/standard/action/Standard-Portal.xml:67`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 206. `/error.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.errorHandle`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:12`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `code` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `message` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 207. `/initUserRegOk.do`

- **请求方法**：`GET/POST`
- **功能说明**：用户注册初始化
- **处理类/方法**：`cn.com.tsg.user.action.UserAction.initUserRegOk`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:19`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `code` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `status` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 208. `/isCodeOk.action`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.admin.action.customPage.CustomAction.isCodeOk`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/customPage/chosePage.xml:7`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 209. `/linkReg.do`

- **请求方法**：`GET/POST`
- **功能说明**：跳转用户注册页面
- **处理类/方法**：`cn.com.tsg.user.action.UserAction.linkReg`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:18`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 210. `/login.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.user.action.UserAction.webLogin`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:14`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 211. `/mobile_updateUserInfo.do`

- **请求方法**：`GET/POST`
- **功能说明**：修改个人信息
- **处理类/方法**：`cn.com.tsg.user.action.UserAction.mobile_updateUserInfo`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:22`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `field` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `value` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 212. `/proxyDebug.do`

- **请求方法**：`GET/POST`
- **功能说明**：-
- **处理类/方法**：`cn.com.tsg.user.action.ApiAction.proxyDebug`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:13`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| - | - | - | 无显式参数或由请求体/过滤器动态处理 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 213. `/userLogin.action`

- **请求方法**：`GET/POST`
- **功能说明**：登录页选择
- **处理类/方法**：`cn.com.tsg.admin.action.customPage.CustomAction.login`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/admin/action/customPage/chosePage.xml:5`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `oauth` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `type` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `redirect_uri` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `SourceHost` | Header/String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 214. `/userRegOk.do`

- **请求方法**：`GET/POST`
- **功能说明**：用户注册 修改失败,TokenId不合法 修改失败,非法操作 真实姓名不能为空 用户名不能为空 密码不能为空 密码的长度为8~16个字符 用户名重复 邮件重复 手机号重复 身份证号重复
- **处理类/方法**：`cn.com.tsg.user.action.UserAction.userRegOk`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:20`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `tokenId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `attr` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `appShowIp` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `accountId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `status` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `gender` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `realName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `email` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `mobile` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `telephone` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `birth` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `position` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `idCard` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workID` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workunit` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `userName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `passwor` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

#### 215. `/userUpdateUserInfo.do`

- **请求方法**：`GET/POST`
- **功能说明**：修改个人信息
- **处理类/方法**：`cn.com.tsg.user.action.UserAction.updateUserInfo`
- **是否在 DispatcherFilter 加载**：是
- **源码/配置位置**：`TrustMore/src/cn/com/tsg/user/action/user-action.xml:21`

**请求参数**

| 参数名 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `gender` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `email` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `mobile` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `telephone` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `birth` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `position` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `idCard` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workID` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `realName` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `workunit` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |
| `deptId` | String | 源码未判定 | 参数名来自源码 `getParameter/getHeader` 静态扫描；必填性需结合分支校验或联调确认 |

**响应说明**

- 返回 XML、JSON、HTML 跳转或文本结果，具体格式由接口实现及 `dataType/dateType` 参数决定。

## 8. 常见返回码

| 返回码 | 含义 |
|---|---|
| `100` | 成功，常见于认证、注销、修改密码等接口 |
| `101` | 参数为空、账号为空、Token 为空或业务校验失败，具体含义依接口而定 |
| `102` | 密码为空、令牌非法、证书验证失败或对象不存在，具体含义依接口而定 |
| `103` - `199` | 业务失败码，常见于账号禁用、吊销、锁定、证书过期、策略不满足等 |
| `200` | 部分接口表示成功，例如 SPA 认证返回 `code=200` |
| `500` | 服务端或业务处理异常 |

## 9. 安全与加密说明

- 密码、随机数、签名、证书等安全参数以实际前端 SDK 和服务端实现为准，联调时应确认是否启用 RSA、国密或其他加密流程。
- 证书参数通常传 Base64 内容；是否包含 PEM 头尾需按具体接口实现确认。
- 生产环境建议启用 HTTPS、请求时间戳、防重放签名、服务端白名单和访问审计。
- 文档中标记为“源码未判定”的参数名来自源码静态扫描；必填性、枚举值和响应结构需以分支校验逻辑及联调结果为准。

## 10. 变更记录

| 版本 | 日期 | 变更说明 |
|:---|:---|:---|
| V1.0 | 2026-05-09 | 由 `API_Documentation.md` 初步整理核心对外接口 |
| V1.1 | 2026-05-09 | 建立模块化索引和交付框架，保留占位区 |
| V1.2 | 2026-05-09 | 合并 `TrustMore_标准对外API文档.md` 与 V1.1，补齐全部 215 条 HTTP 接口详细定义，移除占位区 |

## 11. 交付说明

- 本文档为 V1.2 统一版，后续应以本文件作为对外 API 文档维护基线。
- `/admin/*` 管理后台接口未纳入标准对外 API 交付范围。
- 如需进一步交付给第三方，建议按实际授权范围裁剪模块，并补充真实请求/响应样例。