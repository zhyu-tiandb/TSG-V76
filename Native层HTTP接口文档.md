# TSG6 Native层 HTTP接口详细文档

## 文档说明
- **源文件**: TrustMoreNativeService_TSG6.c
- **协议版本**: TSG 6.0
- **通信协议**: HTTPS (SSL/TLS)
- **请求方式**: GET
- **数据格式**: JSON响应 / XML支持
- **生成时间**: 2025-01-09

---

## 接口概览

| ID | 接口名称 | 描述 | URL路径 |
|----|---------|------|---------|
| 0 | 查询登录类型 | query_login_type | /userportal/authPolicy.action |
| 1 | 密码登录 | login_pwd | /userportal/passLogin.action |
| 2 | 文档证书登录 | login_doc_cert | /userportal/certLogin.action |
| 3 | 密钥证书登录 | login_key_cert | /userportal/usbKLogin.action |
| 4 | 注销 | logout | /userportal/logout.action |
| 5 | 修改密码 | change_password | /userverify/changePwd.action |
| 6 | 获取应用列表 | get-app-list | /userverify/userAppList.action |
| 7 | 查询用户信息 | get-user-info | /userverify/queryUserById.action |
| 8 | 更新用户信息 | update-user-info | /userverify/updateUserById.action |
| 9 | 注册设备 | register-device | /userverify/deviceReg.action |
| 10 | 获取设备列表 | get-device-list | /userverify/deviceList.action |
| 11 | 获取设备信息 | get-device-info | /userverify/getDeviceById.action |
| 12 | 绑定设备 | bind-device | /userverify/setDeviceUser.action |
| 13 | 解绑设备 | unbind-device | /userverify/setDeviceUser.action |
| 14 | 检查设备 | check-device | /userverify/checkDevice.action |
| 15 | 申请绑定设备 | apply_bind-device | /userverify/bindDevice.action |
| 16 | 申请证书 | apply-cert | /userportal/applyUserCert.action |
| 17 | 查询证书状态 | apply-cert-state | /userverify/getUserCertStatus.action |
| 18 | 下载证书 | get-cert-addr-to-download | /userverify/downloadAccountCert.action |
| 19 | 注销证书 | signoff-cert | /userverify/revokeAccountCert.action |
| 20 | 注册用户 | register-user | /userportal/register.action |
| 21 | 验证令牌 | validate-token | /api/hb.do |
| 22 | 查询部门列表 | get-dept-list | /userportal/queryDeptList.action |
| 23 | 检查账号信息 | check-account-info | /userportal/checkAccountInfo.action |
| 24 | 获取账号ID | get-account_id | 本地函数 |
| 25 | 打开应用 | open-app-info | /vd_auth/api/openApp.do |
| 26 | 获取版本 | get-new-version | /userportal/getNewVersion.action |
| 27 | 根据ID查询部门 | get-dept-list-by-id | /userportal/queryDeptByPID.action |
| 28 | 获取在线设备 | get-online-device | /userverify/getOnlineDevice.action |
| 29 | 上传账号头像 | upload-account-icon | /userverify/uploadAccountIcon.action |
| 31 | 获取心跳周期 | get-hbeat-cycle-time | /userverify/getHBeatCycleTime.action |

---

## 详细接口说明

### 一、认证登录类

#### 1. 查询登录类型 (ID: 0)

**描述**: `query_login_type`

**HTTP请求**:
```http
GET /userportal/authPolicy.action HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): isPad - 设备类型标识
- `%s` (第2个): gateAddr - 网关地址
- `%d` (第3个): uni_ssl_port - SSL端口
- `%s` (第4个): g_cookie - Cookie

**处理函数**: `login_type_ops`
- `init`: tsg6_query_login_type_init
- `handler`: tsg6_query_login_type_handler

**使用场景**: 应用启动时查询支持的登录方式

---

#### 2. 密码登录 (ID: 1)

**描述**: `login_pwd`

**HTTP请求**:
```http
GET /userportal/passLogin.action?userName={user}&password={pwd}&dataType=xml HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): user - 用户名（从JSON解析）
- `%s` (第2个): pwd - 密码（从JSON解析）
- `%s` (第3个): isPad
- `%s` (第4个): gateAddr
- `%d` (第5个): uni_ssl_port
- `%s` (第6个): g_cookie

**请求JSON**:
```json
{
  "user": "username",
  "password": "password"
}
```

**处理函数**: `login_pwd_ops`
- `init`: tsg6_query_login_init
- `pre_handler`: tsg6_query_login_pre_handler
- `handler`: tsg6_query_login_pwd_handler
- `post_handler`: tsg6_query_login_post_handler

**后处理**:
- 解析响应获取 `code`, `sessionid`, `tokenid`, `accountId`
- code=100时设置 `isLogin = 1`
- 保存sessionID, tokenID, accountId到全局变量

---

#### 3. 文档证书登录 (ID: 2)

**描述**: `login_doc_cert`

**HTTP请求**:
```http
GET /userportal/certLogin.action?randomStr={random}&userCert={cert}&randomSign={sign}&password={pwd} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): random - URL编码的随机数
- `%s` (第2个): cert - URL编码的证书
- `%s` (第3个): sign - URL编码的签名
- `%s` (第4个): pwd - URL编码的密码
- `%s` (第5个): isPad
- `%s` (第6个): gateAddr
- `%d` (第7个): uni_ssl_port
- `%s` (第8个): g_cookie

**请求JSON**:
```json
{
  "random": "timestamp||UUID",
  "cert": "certificate_content",
  "sign": "signature",
  "pwd": "password"
}
```

**处理函数**: `login_doccert_ops`
- `handler`: tsg6_query_login_doccert_handler

**URL编码**: 所有参数都需要URL编码

---

#### 4. 密钥证书登录 (ID: 3)

**描述**: `login_key_cert`

**HTTP请求**:
```http
GET /userportal/usbKLogin.action?randomStr={random}&randomSign={sign}&userCert={cert} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): random - URL编码的随机数
- `%s` (第2个): sign - URL编码的签名
- `%s` (第3个): cert - URL编码的证书
- `%s` (第4个): isPad
- `%s` (第5个): gateAddr
- `%d` (第6个): uni_ssl_port
- `%s` (第7个): g_cookie

**请求JSON**:
```json
{
  "random": "timestamp||UUID",
  "cert": "certificate_content",
  "sign": "signature"
}
```

**处理函数**: `login_keycert_ops`
- `handler`: tsg6_query_login_keycert_handler

**支持硬件**: USB Key、TF卡（海泰、龙马等）

---

### 二、账号管理类

#### 5. 注销登录 (ID: 4)

**描述**: `logout`

**HTTP请求**:
```http
GET /userportal/logout.action?tokenId={tokenID} HTTP/1.1
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): tokenID - 认证令牌
- `%s` (第2个): isPad
- `%s` (第3个): gateAddr
- `%d` (第4个): uni_ssl_port
- `%s` (第5个): g_cookie

**处理函数**: `logout_ops`
- `post_handler`: 清空应用列表(al_6)、清空tokenID/sessionID

---

#### 6. 修改密码 (ID: 5)

**描述**: `change_password`

**HTTP请求**:
```http
GET /userverify/changePwd.action?oldpassword={oldpwd}&newpassword={newpwd} HTTP/1.1
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): oldpwd - 旧密码
- `%s` (第2个): newpwd - 新密码
- `%s` (第3个): isPad
- `%s` (第4个): gateAddr
- `%d` (第5个): uni_ssl_port
- `%s` (第6个): g_cookie

**请求JSON**:
```json
{
  "oldpwd": "old_password",
  "newpwd": "new_password"
}
```

**处理函数**: `changepwd_ops`

---

#### 7. 验证令牌 (ID: 21)

**描述**: `validate-token`

**HTTP请求**:
```http
GET /api/hb.do?tokenId={tokenID} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): tokenID
- `%s` (第2个): isPad
- `%s` (第3个): gateAddr
- `%d` (第4个): uni_ssl_port
- `%s` (第5个): g_cookie

**处理函数**: `validateToken_ops`

**用途**: 心跳检测、会话有效性验证

---

#### 8. 检查账号信息 (ID: 23)

**描述**: `check-account-info`

**HTTP请求**:
```http
GET /userportal/checkAccountInfo.action?accountId={accountId}&data={data}&type={type} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): accountId - 账号ID
- `%s` (第2个): data - 检查的数据（用户名/邮箱等）
- `%s` (第3个): type - 检查类型
- `%s` (第4个): isPad
- `%s` (第5个): gateAddr
- `%d` (第6个): uni_ssl_port
- `%s` (第7个): g_cookie

**请求JSON**:
```json
{
  "data": "username_or_email",
  "type": "username"
}
```

**处理函数**: `checkAccountInfo_ops`

---

#### 9. 获取账号ID (ID: 24)

**描述**: `get-account_id`

**类型**: `TSG6_TSG_LOCAL_FUNCTION` (本地函数)

**处理方式**: 直接返回accountId整数值的字符串形式

**处理函数**: `getaccountid_ops`
- `handler`: `sprintf(httppost, "%d", accountId);`

**用途**: 获取当前登录账号的ID

---

### 三、用户信息类

#### 10. 查询用户信息 (ID: 7)

**描述**: `get-user-info`

**HTTP请求**:
```http
GET /userverify/queryUserById.action?accountId={accountId} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): accountId
- `%s` (第2个): isPad
- `%s` (第3个): gateAddr
- `%d` (第4个): uni_ssl_port
- `%s` (第5个): g_cookie

**处理函数**: `getUserInfo_ops`

---

#### 11. 更新用户信息 (ID: 8)

**描述**: `update-user-info`

**HTTP请求**:
```http
GET /userverify/updateUserById.action?accountId={accountId}&{userInfo} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): accountId
- `%s` (第2个): userInfo - URL编码的用户信息字符串
- `%s` (第3个): isPad
- `%s` (第4个): gateAddr
- `%d` (第5个): uni_ssl_port
- `%s` (第6个): g_cookie

**请求JSON**:
```json
{
  "userInfo": "name=value&name2=value2..."
}
```

**处理函数**: `updateUserInfo_ops`

---

### 四、应用管理类

#### 12. 获取应用列表 (ID: 6)

**描述**: `get-app-list`

**HTTP请求**:
```http
GET /userverify/userAppList.action?accountId={accountId}&tokenId={tokenID} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): accountId
- `%s` (第2个): tokenID
- `%s` (第3个): isPad
- `%s` (第4个): gateAddr
- `%d` (第5个): uni_ssl_port
- `%s` (第6个): g_cookie

**处理函数**: `getAppList_ops`
- `pre_handler`: 清空旧应用列表，重置dataChannel为8001
- `post_handler`: 解析JSON响应，填充al_6应用列表

**响应解析**:
```json
{
  "content": {
    "CsApp": [
      {
        "id": "app001",
        "appAddr": "192.168.1.100",
        "Port": "8080",
        "IsTCP": 1,
        "tcpType": "TCP"或"HTTP",
        "isCompress": 0或1
      }
    ]
  }
}
```

**端口格式支持**:
- 单端口: `8080`
- 逗号分隔: `8080,8081,8082`
- 连续范围: `8080-8090`

**本地端口计算**: `listen_port = 8000 + dst_port`

---

#### 13. 打开应用 (ID: 25)

**描述**: `open-app-info`

**HTTP请求**:
```http
GET /vd_auth/api/openApp.do?tokenId={tokenID}&id={id}&type={type} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): tokenID
- `%s` (第2个): id - 应用ID
- `%s` (第3个): type - 应用类型
- `%s` (第4个): isPad
- `%s` (第5个): gateAddr
- `%d` (第6个): uni_ssl_port
- `%s` (第7个): g_cookie

**请求JSON**:
```json
{
  "id": "app001",
  "type": "cs"
}
```

**处理函数**: `openapp_ops`

---

### 五、设备管理类

#### 14. 注册设备 (ID: 9)

**描述**: `register-device`

**HTTP请求**:
```http
GET /userverify/deviceReg.action?tokenId={tokenID}&dataType=xml&{deviceInfo} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): tokenID
- `%s` (第2个): deviceInfo - URL编码的设备信息
- `%s` (第3个): isPad
- `%s` (第4个): gateAddr
- `%d` (第5个): uni_ssl_port
- `%s` (第6个): g_cookie

**请求JSON**:
```json
{
  "deviceInfo": "deviceId=xxx&deviceName=xxx..."
}
```

**处理函数**: `registerDev_ops`

---

#### 15. 获取设备列表 (ID: 10)

**描述**: `get-device-list`

**HTTP请求**:
```http
GET /userverify/deviceList.action?deviceState={deviceState}&accountId={accountId} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): deviceState - 设备状态（固定为3）
- `%d` (第2个): accountId
- `%s` (第3个): isPad
- `%s` (第4个): gateAddr
- `%d` (第5个): uni_ssl_port
- `%s` (第6个): g_cookie

**处理函数**: `getDevList_ops`

---

#### 16. 获取设备信息 (ID: 11)

**描述**: `get-device-info`

**HTTP请求**:
```http
GET /userverify/getDeviceById.action?id={devId} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): devId - 设备ID
- `%s` (第2个): isPad
- `%s` (第3个): gateAddr
- `%d` (第4个): uni_ssl_port
- `%s` (第5个): g_cookie

**请求JSON**:
```json
{
  "devId": "device001"
}
```

**处理函数**: `getDevInfo_ops`

---

#### 17. 绑定设备 (ID: 12)

**描述**: `bind-device`

**HTTP请求**:
```http
GET /userverify/setDeviceUser.action?accountId={accountId}&deviceId={devId}&State={State} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): accountId
- `%s` (第2个): devId
- `%d` (第3个): State - 状态（固定为1表示绑定）
- `%s` (第4个): isPad
- `%s` (第5个): gateAddr
- `%d` (第6个): uni_ssl_port
- `%s` (第7个): g_cookie

**请求JSON**:
```json
{
  "devId": "device001"
}
```

**处理函数**: `bindDev_ops`

---

#### 18. 解绑设备 (ID: 13)

**描述**: `unbind-device`

**HTTP请求**:
```http
GET /userverify/setDeviceUser.action?accountId={accountId}&deviceId={devId}&State={State} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): accountId
- `%s` (第2个): devId
- `%d` (第3个): State - 状态（固定为0表示解绑）
- `%s` (第4个): isPad
- `%s` (第5个): gateAddr
- `%d` (第6个): uni_ssl_port
- `%s` (第7个): g_cookie

**请求JSON**:
```json
{
  "devId": "device001"
}
```

**处理函数**: `unbindDev_ops`

**注意**: 与绑定设备使用相同的URL，通过State参数区分

---

#### 19. 检查设备 (ID: 14)

**描述**: `check-device`

**HTTP请求**:
```http
GET /userverify/checkDevice.action?tokenId={tokenID}&dataType=xml&{deviceInfo} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): tokenID
- `%s` (第2个): deviceInfo - URL编码的设备信息
- `%s` (第3个): isPad
- `%s` (第4个): gateAddr
- `%d` (第5个): uni_ssl_port
- `%s` (第6个): g_cookie

**请求JSON**:
```json
{
  "deviceInfo": "deviceId=xxx..."
}
```

**处理函数**: `checkDev_ops`

---

#### 20. 申请绑定设备 (ID: 15)

**描述**: `apply_bind-device`

**HTTP请求**:
```http
GET /userverify/bindDevice.action?deviceType=2&accountId={accountId}&{deviceInfo} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): accountId
- `%s` (第2个): deviceInfo - URL编码的设备信息
- `%s` (第3个): isPad
- `%s` (第4个): gateAddr
- `%d` (第5个): uni_ssl_port
- `%s` (第6个): g_cookie

**请求JSON**:
```json
{
  "deviceInfo": "deviceId=xxx..."
}
```

**处理函数**: `applyBindDev_ops`

**注意**: deviceType固定为2

---

#### 21. 获取在线设备 (ID: 28)

**描述**: `get-online-device`

**HTTP请求**:
```http
GET /userverify/getOnlineDevice.action?tokenId={tokenID}&accountId={accountId}&excludeLocal={excludeLocal} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): tokenID
- `%d` (第2个): accountId
- `%s` (第3个): excludeLocal - 是否排除本地设备
- `%s` (第4个): isPad
- `%s` (第5个): gateAddr
- `%d` (第6个): uni_ssl_port
- `%s` (第7个): g_cookie

**请求JSON**:
```json
{
  "excludeLocal": "true"
}
```

**处理函数**: `getonlinedevice_ops`

---

### 六、证书管理类

#### 22. 申请证书 (ID: 16)

**描述**: `apply-cert`

**HTTP请求**:
```http
GET /userportal/applyUserCert.action?accountId={accountId}&CertRequest={CertRequest}&CertPass={CertPass}&certType={certType} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): accountId
- `%s` (第2个): CertRequest - 证书请求（固定为"SAG"）
- `%s` (第3个): CertPass - 证书密码
- `%s` (第4个): certType - 证书类型（固定为"sign"）
- `%s` (第5个): isPad
- `%s` (第6个): gateAddr
- `%d` (第7个): uni_ssl_port
- `%s` (第8个): g_cookie

**请求JSON**:
```json
{
  "CertPass": "password"
}
```

**处理函数**: `applyCert_ops`

**注意**: CertRequest和certType是固定值

---

#### 23. 查询证书状态 (ID: 17)

**描述**: `apply-cert-state`

**HTTP请求**:
```http
GET /userverify/getUserCertStatus.action?accountId={accountId} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): accountId
- `%s` (第2个): isPad
- `%s` (第3个): gateAddr
- `%d` (第4个): uni_ssl_port
- `%s` (第5个): g_cookie

**处理函数**: `queryCertState_ops`

---

#### 24. 下载证书 (ID: 18)

**描述**: `get-cert-addr-to-download`

**HTTP请求**:
```http
GET /userverify/downloadAccountCert.action?accountId={accountId} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): accountId
- `%s` (第2个): isPad
- `%s` (第3个): gateAddr
- `%d` (第4个): uni_ssl_port
- `%s` (第5个): g_cookie

**处理函数**: `downloadCertAddr_ops`

---

#### 25. 注销证书 (ID: 19)

**描述**: `signoff-cert`

**HTTP请求**:
```http
GET /userverify/revokeAccountCert.action?accountId={accountId} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%d` (第1个): accountId
- `%s` (第2个): isPad
- `%s` (第3个): gateAddr
- `%d` (第4个): uni_ssl_port
- `%s` (第5个): g_cookie

**处理函数**: `signoffCert_ops`

---

### 七、组织架构类

#### 26. 查询部门列表 (ID: 22)

**描述**: `get-dept-list`

**HTTP请求**:
```http
GET /userportal/queryDeptList.action HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): isPad
- `%s` (第2个): gateAddr
- `%d` (第3个): uni_ssl_port
- `%s` (第4个): g_cookie

**处理函数**: `querydeptlist_ops`

---

#### 27. 根据ID查询部门 (ID: 27)

**描述**: `get-dept-list-by-id`

**HTTP请求**:
```http
GET /userportal/queryDeptByPID.action?deptId={deptId} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): deptId - 部门ID
- `%s` (第2个): isPad
- `%s` (第3个): gateAddr
- `%d` (第4个): uni_ssl_port
- `%s` (第5个): g_cookie

**请求JSON**:
```json
{
  "deptID": "dept001"
}
```

**处理函数**: `querydeptlistbyid_ops`

---

### 八、用户注册类

#### 28. 注册用户 (ID: 20)

**描述**: `register-user`

**HTTP请求**:
```http
GET /userportal/register.action?{userInfo} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): userInfo - URL编码的用户信息字符串
- `%s` (第2个): isPad
- `%s` (第3个): gateAddr
- `%d` (第4个): uni_ssl_port
- `%s` (第5个): g_cookie

**请求JSON**:
```json
{
  "userInfo": "userName=xxx&loginName=xxx&password=xxx..."
}
```

**处理函数**: `registerUser_ops`
- `pre_handler`: 重置accountId为0
- `post_handler`: 从响应中解析并保存accountId

---

### 九、系统功能类

#### 29. 获取版本 (ID: 26)

**描述**: `get-new-version`

**HTTP请求**:
```http
GET /userportal/getNewVersion.action HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): isPad
- `%s` (第2个): gateAddr
- `%d` (第3个): uni_ssl_port
- `%s` (第4个): g_cookie

**处理函数**: `getversion_ops`

---

#### 30. 获取心跳周期 (ID: 31)

**描述**: `get-hbeat-cycle-time`

**HTTP请求**:
```http
GET /userverify/getHBeatCycleTime.action?clientType=1&tokenId={tokenID} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): tokenID
- `%s` (第2个): isPad
- `%s` (第3个): gateAddr
- `%d` (第4个): uni_ssl_port
- `%s` (第5个): g_cookie

**处理函数**: `gethbeatcycletime_ops`

**注意**: URL中clientType=1是固定的（缺少&符号）

---

#### 31. 上传账号头像 (ID: 29)

**描述**: `upload-account-icon`

**HTTP请求**:
```http
GET /userverify/uploadAccountIcon.action?tokenId={tokenID}&accountId={accountId}&iconData={iconData} HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Android {isPad}
Host: {gateAddr}:{uni_ssl_port}
Cookie: {g_cookie}
```

**参数说明**:
- `%s` (第1个): tokenID
- `%d` (第2个): accountId
- `%s` (第3个): iconData - Base64编码的图片数据
- `%s` (第4个): isPad
- `%s` (第5个): gateAddr
- `%d` (第6个): uni_ssl_port
- `%s` (第7个): g_cookie

**请求JSON**:
```json
{
  "iconData": "base64_encoded_image_data"
}
```

**处理函数**: `uploadaccounticon_ops`

---

## 全局变量说明

### 认证相关
```c
static int accountId;          // 账号ID
static char tokenID[256];      // 令牌ID
static char sessionID[256];    // 会话ID
static int isLogin;            // 登录状态
```

### 配置相关
```c
static char isPad[10];         // 设备类型标识
static char gateAddr[64];      // 网关地址
static int uni_ssl_port;       // SSL端口
static char g_cookie[512];     // Cookie
```

### 应用列表
```c
struct app_list *al_6;         // 应用列表（TSG6格式）
```

---

## SSL/TLS配置

### 支持的加密套件
```c
// 非国密
int ciphersuites[] = { TLS_RSA_WITH_AES_128_CBC_SHA, 0 };

// 国密
int ciphersuites[] = { ECC_SM4_SM3, 0 };
```

### 国密支持
- 当 `cipherType == 1` 时使用国密算法
- 使用SM4加密、SM3签名
- SSL版本设置为1.1

### SSL初始化流程
1. 调用 `ssl_init()` 初始化SSL上下文
2. 配置随机数生成器
3. 建立TCP连接 `net_connect()`
4. 设置加密套件
5. 执行SSL握手 `ssl_handshake()`

---

## 响应处理

### JSON验证
```c
static int isJson(char* str) {
    cJSON *pRoot = cJSON_Parse(str);
    if (pRoot == NULL) {
        return 0;  // 不是有效JSON
    }
    cJSON_Delete(pRoot);
    return 1;  // 是有效JSON
}
```

### 错误响应格式
```json
{
  "state": "FAIL",
  "message": "invalid message:..."
}
```

---

## 工具函数

### URL编码
```c
char *url_encode(char *str, int len, int *outLen);
```

用于对请求参数进行URL编码，主要在证书登录接口中使用。

---

## 请求流程

1. **初始化阶段**
   - 调用 `init()` 函数（如果存在）
   - 准备请求参数

2. **预处理阶段**
   - 调用 `pre_handler()` 函数（如果存在）
   - 清空旧数据、初始化状态

3. **请求处理阶段**
   - 调用 `handler()` 函数
   - 构建HTTP请求字符串
   - 执行SSL通信

4. **后处理阶段**
   - 调用 `post_handler()` 函数（如果存在）
   - 解析响应数据
   - 更新全局状态

---

## 与Java层交互

### JNI函数
```c
JNIEXPORT jstring JNICALL Java_com_tsg_service_ITrustMoreNativeService_tsg6request(
    JNIEnv *env, 
    jobject obj, 
    jint requestid,      // 请求ID
    jstring jsonPara     // JSON参数
);
```

### 设置设备类型
```c
JNIEXPORT jboolean JNICALL Java_com_tsg_service_ITrustMoreNativeService_setPad(
    JNIEnv *env, 
    jobject obj, 
    jstring setPad       // "Phone" 或 "Pad"
);
```

---

## 注意事项

1. **参数顺序**: sprintf参数顺序必须与HTTP请求模板中的占位符顺序一致
2. **URL编码**: 证书登录相关接口的所有参数都需要URL编码
3. **Cookie管理**: 登录接口会设置Cookie，后续请求携带
4. **端口格式**: 应用端口支持三种格式（单端口、逗号分隔、范围）
5. **本地函数**: ID 24是本地函数，不发起网络请求
6. **SSL重用**: 每次请求都建立新的SSL连接
7. **错误处理**: 网络错误返回固定JSON格式的错误响应
8. **JSON验证**: 所有响应都验证是否为有效JSON格式

---

## 文档信息
- **源文件**: module_service/src/jni/TrustMoreNativeService_TSG6.c
- **行数**: 1492行
- **接口数量**: 31个
- **文档生成**: 2025-01-09
- **适用项目**: TrustMore安全网关 Android客户端 V2.3.0.33
