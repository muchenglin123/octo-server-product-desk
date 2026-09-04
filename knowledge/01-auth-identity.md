# octo-server 知识卡片：认证与身份

## 1. 考官可能问什么

1. octo-server 的用户态 HTTP 认证 token 放在哪里？支持 `Authorization: Bearer` 吗？
2. cookie 是登录凭据吗？
3. Bot token 和用户 token 是不是同一套？
4. App Bot 的 API token 和 WebSocket / IM token 是不是同一个？
5. daemon / API key（`uk_`）如何鉴权？
6. OIDC / OAuth 登录、`/exchange`、`/exchange-jwt` 分别处理什么？
7. webhook token 和 webhook 签名分别是什么边界？

## 2. 一句话结论

octo-server 的认证不是单一 token 体系，而是多套凭据并存：用户 HTTP 会话 token、Bearer 兼容、扫码登录/OIDC/OAuth 登录、Bot token、`uk_` User API Key、incoming webhook URL token，以及 webhook HMAC 签名各自服务不同入口。

## 3. 产品解释

- 人类用户登录后拿到 session token，后续业务 API 主要靠 `token` header；标准 `Authorization: Bearer` 会被兼容层回填为 `token`。
- cookie 当前源码证据显示用于语言协商，不是登录态主凭据。
- Bot API 走 Bearer token，`app_` 前缀代表 App Bot，其他/`bf_` 代表 User Bot。
- Bot 注册会返回 IM 连接信息；User Bot 复用 bot token 作为 IM token，App Bot 也明确复用 App token 作为 API + IM WebSocket token。
- `uk_` API key 是 daemon / bot provision 场景的外部凭据，校验方式不同于用户 session token。
- OIDC/OAuth 是外部身份提供方接入层，最终换成本服务会话。
- incoming webhook 使用 URL path token，但服务端只存 hash 并做日志脱敏；webhook 回调签名另走 HMAC。

## 4. 源码证据

1. 用户 HTTP 认证主入口是 `token` header；`Authorization: Bearer` 只是兼容层，且 `token` header 优先、不接受 query token。  
   来源: `pkg/wkhttp/bearer_compat.go#L17-L43`  
   来源: `pkg/wkhttp/bearer_compat.go#L44-L75`

2. token 解析会从 session store / cache 读取并解码；空 token、找不到 token、非法 token 分别处理。角色会按 resolver 读权威状态，避免只信 token 快照。  
   来源: `pkg/auth/parser.go#L133-L165`  
   来源: `pkg/auth/parser.go#L189-L224`

3. 用户登录会生成 UUID token；APP 登录撤销旧 token，PC/Web 类设备可复用旧 token；随后调用 IM 更新 token。  
   来源: `modules/user/api.go#L1841-L1877`  
   来源: `modules/user/api.go#L1883-L1924`

4. Session Store 用 Redis `SETNX` 写入新 token，避免碰撞；复用旧 token 时不延长 deadline，且缺失的旧 token 不会被重新创建。  
   来源: `pkg/auth/session_store.go#L269-L303`  
   来源: `pkg/auth/session_store.go#L306-L325`

5. cookie 参与语言协商：优先级是可信 `X-Octo-Lang`、URL `lang`、cookie `i18n_lang`、用户语言、`Accept-Language`、默认语言；这里不是认证 cookie。  
   来源: `pkg/i18n/lang.go#L50-L83`

6. Bot API 认证从 `Authorization: Bearer` 取 token；`app_` 走 App Bot，其他/`bf_` 走 User Bot。  
   来源: `modules/bot_api/auth.go#L25-L42`  
   来源: `modules/bot_api/auth.go#L143-L150`

7. User Bot token 查 `robot` 表；App Bot token 先查共享 registry/cache，miss 后查 DB，且 App Bot 必须 `status=1` 才能服务 API。  
   来源: `modules/bot_api/auth.go#L45-L61`  
   来源: `modules/bot_api/auth.go#L64-L97`

8. Bot 注册接口返回 `im_token` 与 `ws_url`；User Bot 复用 `bot_token` 作为 IM token。  
   来源: `modules/bot_api/register.go#L47-L67`  
   来源: `modules/bot_api/register.go#L104-L121`

9. App Bot 明确使用同一个 token 做 API 认证和 IM WebSocket 连接。  
   来源: `modules/bot_api/register.go#L143-L173`

10. daemon 获取 bot token 的接口要求 `Authorization: Bearer <key>`，并要求 bot 存在且启用、调用者是 bot creator、bot 属于 API key 绑定 Space。  
    来源: `modules/bot_provision/bot_api.go#L96-L108`  
    来源: `modules/bot_provision/bot_api.go#L109-L171`

11. `uk_` API key 解析会查 `user_api_key`，要求 `space_id!=''`、`status=1`、`client_id='botfather'`，并再次校验 key owner 仍是 Space 成员。  
    来源: `modules/bot_provision/resolve.go#L61-L84`

12. OIDC 路由包含 authorize/callback/exchange/exchange-jwt/logout；logout 挂用户认证，exchange 类用于外部 token 换本地登录态。  
    来源: `modules/oidc/api.go#L360-L427`

13. `/exchange-jwt` 会读取并校验 JWT，失败统一按未授权处理，避免枚举差异。  
    来源: `modules/oidc/api_exchange_jwt.go#L40-L116`

14. incoming webhook 以 path token 找 webhook，服务端只存 token hash，并做常量时间比较。  
    来源: `modules/incomingwebhook/api.go#L1302-L1340`

15. webhook 签名使用 HMAC-SHA256，并依赖环境中的 webhook secret。  
    来源: `modules/webhook/hmac.go#L15-L58`

## 5. 边界与易错点

- 不要说 cookie 是登录主凭据；当前可核验证据里 cookie 用于语言协商。
- 不要混淆用户 token、Bot token、App Bot token、`uk_` API key，它们的鉴权入口和作用域不同。
- WebSocket / IM token 在 Bot 注册响应中出现，不能等同于用户 HTTP session token。
- 目标仓库只读；任何密钥、token、cookie 都不能出现在群聊或 GitHub 文件里。

## 6. 考试可直接回答

> 结论：octo-server 的认证路径是多套并存。用户业务 API 主要走 `token` header，并兼容 `Authorization: Bearer`；Bot API 也走 Bearer，但按 `app_` 与 `bf_`/legacy token 分为 App Bot 和 User Bot；daemon 场景使用 `uk_` API key；OIDC/OAuth 用外部身份换本地登录态；incoming webhook 使用 URL token + hash 比对，内部 webhook/GitHub webhook 走 HMAC 签名。cookie 目前只看到用于语言协商，不是登录态主凭据。

## 7. 待补 / 不确定项

- WebSocket 用户端握手若考官追问，需要继续查 WuKongIM 侧协议与 octo-server 对接点。
