# octo-server 知识卡片：Bot 与 Agent

## 1. 考官可能问什么

1. octo-server 里 Bot 分哪几类？User Bot 和 App Bot 有什么区别？
2. BotFather 现在还负责 `/v1/bot/*` 吗？
3. Bot API 怎么识别 `bf_` 和 `app_` token？
4. App Bot 如何创建、发布、发现、申请使用？
5. Agent 如何拿到 API Key / bot token / IM token？
6. botidentity 模块为什么不看 `user.robot`？
7. bot_provision 和 botfather 的边界是什么？

## 2. 一句话结论

octo-server 的 Bot 体系是 User Bot 与 App Bot 双轨：User Bot 属于用户，主要由 botfather 管理；App Bot 属于平台/Space，主要由 app_bot 管理；bot_api 负责统一 Bot API 鉴权和消息能力；bot_provision 给外部 daemon/fleet 提供安全取 token 的桥；botidentity 提供权威 Bot 身份解析。

## 3. 产品解释

- **User Bot**：用户自己的机器人，记录在 `robot` 表，常见 token 为 `bf_` 或 legacy bot token。
- **App Bot**：平台或 Space 发布的应用型机器人，token 以 `app_` 开头，作用域区分 platform / space。
- **BotFather**：偏用户 Bot 管理、API Key、申请审批、运行时 onboarding；主 Bot API 迁到 `bot_api`。
- **bot_api**：Bot 调用服务端能力的统一入口，负责认证、发消息、群/线程、卡片、事件、文件等能力。
- **bot_provision**：为拆分后的 daemon/fleet 提供“用 `uk_` API Key 拉取 bot token”的受控接口。
- **botidentity**：统一判断某个 UID 是否活跃 Bot，读取 `robot` 与 `app_bot` 权威来源。

## 4. 源码证据

1. App Bot 有独立 token/UID 命名规则，`app_` token、`app_<id>_bot` UID，并保留部分系统 ID。  
   来源: `modules/app_bot/app_bot.go#L30-L60`

2. App Bot 模块启动时注册 App Bot 身份解析器，并初始化共享 Redis auth registry，让 token 撤销跨副本立即生效。  
   来源: `modules/app_bot/app_bot.go#L83-L101`

3. App Bot 管理 API 分 platform admin 和 space admin 两组；另有用户发现 `/v1/app_bot/available` 与用户申请 `/v1/app_bot/apply`。  
   来源: `modules/app_bot/app_bot.go#L116-L153`

4. Space App Bot 管理要求登录用户在该 Space 中 role ≥ admin；路由还会校验 bot 是否属于当前 route scope，避免平台路由读取空间 bot token。  
   来源: `modules/app_bot/app_bot.go#L156-L183`

5. 创建 App Bot 时生成 UID/token，写入 `app_bot`，注册 IM token，并创建 `user` 记录且 `Robot: 1`。  
   来源: `modules/app_bot/app_bot.go#L328-L431`

6. 发布 App Bot 会写入本地 registry 和共享 auth registry；下架会从本地 registry 和 auth registry 移除。  
   来源: `modules/app_bot/app_bot.go#L835-L890`  
   来源: `modules/app_bot/app_bot.go#L923-L938`

7. App Bot 可发现列表显式排除 token；platform bot 对所有人可见，space bot 只对同 Space 成员可见。  
   来源: `modules/app_bot/db.go#L159-L188`

8. 用户申请 App Bot 时，必须传 `app_*_bot` 形态 UID；bot 必须已发布；Space bot 会校验申请人仍是该 Space 成员。  
   来源: `modules/app_bot/app_bot.go#L1031-L1083`

9. Bot API 的统一鉴权按 token 前缀分流：`app_` 走 App Bot，其他走 User Bot；上下文里写入 bot kind、robot id、App Bot scope/space。  
   来源: `modules/bot_api/auth.go#L10-L42`

10. User Bot 通过 `robot.bot_token` 鉴权；App Bot 优先查共享 registry，miss 后查 DB，且 App Bot 必须 `status=1` 才能服务 API 请求。  
    来源: `modules/bot_api/auth.go#L45-L61`  
    来源: `modules/bot_api/auth.go#L64-L129`

11. Bot API 主路由 `/v1/bot` 提供发送消息、事件、消息同步、群/线程、文件、卡片、用户信息、语音上下文、OBO grant 等能力，并统一挂鉴权和限流。  
    来源: `modules/bot_api/bot_api.go#L377-L468`

12. Bot 注册接口 `/v1/bot/register` 对 User Bot 使用 bot token 作为 IM token，并可上报 Agent 平台、Agent 版本、插件版本。  
    来源: `modules/bot_api/register.go#L47-L121`

13. App Bot register 使用同一个 App token 作为 API 和 IM WebSocket token。  
    来源: `modules/bot_api/register.go#L143-L173`

14. bot_provision 允许 daemon 用 `uk_` API key 获取 bot token，但要求调用者是 bot creator，且 bot 属于 API key 绑定 Space。  
    来源: `modules/bot_provision/bot_api.go#L96-L171`

15. Runtime onboarding 会创建或复用 User API Key，并生成 daemon 启动命令。  
    来源: `modules/botfather/api_runtime_onboarding.go#L46-L160`

## 5. 边界与易错点

- BotFather 不等于 Bot API 主实现；主 `/v1/bot/*` 能力在 `bot_api`。
- App Bot token 与 User Bot token 不是同一类；`app_` 前缀是关键分流依据。
- App Bot 有平台/Space 作用域，不能简单说“所有用户都能用所有 App Bot”。
- bot_provision 是外部 daemon/fleet 的桥，不是普通用户登录接口。

## 6. 考试可直接回答

> 结论：octo-server 的 Bot 与 Agent 体系是 User Bot / App Bot 双轨。User Bot 由用户创建，主要由 botfather 管理，使用 `bf_` 或 legacy token；App Bot 由平台或 Space 管理员发布，使用 `app_` token。Bot API 统一接收 Bearer token 并按前缀分流。Agent/daemon 侧则通过注册和 bot_provision 拿到 API URL、IM token、WebSocket URL 或 bot token，再连接 Octo。

## 7. 待补 / 不确定项

- 如果考官追问“Agent runtime 会话怎么调度”，需补 octo-fleet 侧资料；当前 octo-server 内部已注明 runtime/bot orchestration 移出。
