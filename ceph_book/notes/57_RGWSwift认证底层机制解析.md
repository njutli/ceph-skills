# RGW Swift 认证底层机制解析

> 本节分析 OpenStack Swift 兼容模式下的认证实现，重点解析 RGW 如何验证初始请求以及如何生成安全令牌。

## 疑问记录
**问题**: 书中提到“Swift 使用令牌（Token）完成操作请求的验证”，既然操作请求是基于令牌的，那**申请令牌**时 RGW 怎么检验申请令牌的请求？

**分析**: 申请令牌是“信任根（Root of Trust）”的建立过程。虽然操作请求（GET/PUT Object）只认 Token，但在获取 Token 的第一步，必须依赖用户预先持有的 **Secret Key**。

**验证流程**:
1.  客户端向 RGW 请求令牌（通常请求 `/auth` 路径），需在 HTTP Header 中携带以下信息：
    *   `X-Auth-User`: 子用户 ID。
    *   `X-Auth-Key`: 子用户密钥/密码。
2.  RGW 收到请求后，以 `X-Auth-User` 为索引从 RADOS 集群读取用户信息 (`RGWUserInfo`)。
3.  RGW 从用户信息中找到对应的 **Swift Key**，并与请求中携带的 `X-Auth-Key` 进行**明文比对**。
4.  **只有当两者完全一致时**，RGW 才会认为申请请求合法，进而计算签名并生成令牌返回。

---

## 1. 认证入口与请求格式
*   **入口 Handler**: `RGWRESTMgr_SWIFT_Auth` -> `RGW_SWIFT_Auth_Get` (位于 `rgw_appmain.cc`)。
*   **关键请求头**:
    *   `X-Auth-User`: 子用户名 (`tenant:user` 格式)。
    *   `X-Auth-Key`: 对应子用户的私钥/密码。

## 2. 验证逻辑实现
代码位于 `src/rgw/rgw_swift_auth.cc` 的 `RGW_SWIFT_Auth_Get::execute` 函数中：
1.  **提取凭据**: 从 HTTP header 获取 User 和 Key。
2.  **检索用户**: 调用 `driver->get_user_by_swift` 从存储层读取 `RGWUserInfo`。
3.  **密钥比对**: 遍历用户的 `swift_keys` 列表，查找匹配 User 的记录。
    *   **核心校验**: 将请求携带的 Key 与数据库中存储的 Swift Key 进行明文比对 (`string compare`)。
    *   如果 Key 不匹配，返回 `401 Unauthorized` (`-EPERM`)。

## 3. Token 生成机制
认证通过后，RGW 使用 **HMAC-SHA1** 算法生成签名令牌。

*   **函数**: `rgw::auth::swift::encode_token` (调用内部的 `build_token`)。
*   **载荷 (Payload)**:
    1.  `swift_user`: 用户 ID。
    2.  `nonce`: 随机数 (防重放攻击)。
    3.  `expiration`: 过期时间 (默认 24h, 配 `rgw_swift_token_expiration`)。
    4.  `signature`: HMAC(`swift_key`, 数据)。
*   **最终格式**: `AUTH_rgwtk` + 十六进制字符串。

## 4. 后续请求验证
当客户端在后续请求中使用该 Token 时，`SignedTokenEngine` 会：
1.  解码 Token，验证过期时间。
2.  重新计算 HMAC 签名并与 Token 中的签名比对 (验证密钥是否发生变更)。
3.  **优势**: 实现了轻量级验证，无需重新比对密码，且包含有效期管理。
