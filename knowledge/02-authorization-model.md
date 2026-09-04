# 02 鉴权模型

## 1. 考官可能问什么

1. octo-server 的权限模型是不是只有一种 RBAC？
2. 管理后台角色、Space 管理员、频道成员权限、Bot 权限分别管什么？
3. User API Key（`uk_*`）和 Bot token（`bf_*` / `app_*`）的授权边界有什么区别？
4. App Bot 为什么是 DM-only？Space App Bot 如何防跨租户？
5. OBO（on_behalf_of）能不能绕过 Bot 原本的频道权限？
6. Agent 绑定 Bot 如何避免多个 Agent 抢同一个 Bot？
7. 权限不足时如何 fail-closed？

## 2. 一句话结论

octo-server 的鉴权模型是多层叠加：登录态/系统角色负责管理面入口，Space membership/role 负责租户与空间管理员边界，群/子区成员关系负责频道访问，Bot API 按 token 类型和 bot kind 做专用门禁，User API Key 把自动化调用固定到 key 绑定的 Space；这些边界不是互相替代，而是在各自入口按“先认证、再租户/对象权限、再业务动作权限”的顺序共同生效。

## 3. 产品解释

- **系统角色 / 管理面角色**：用于平台管理、管理员控制台等入口。普通用户登录不等于管理员；部分固定角色只开放特定管理面，不自动获得所有 admin API。
- **Space 权限**：Space 成员关系和角色决定用户是否能管理 Space 级 App Bot、是否能把 Bot 加入某个 Space、`uk_*` API Key 是否可访问某 Space 的资源。
- **频道权限**：群、子区、DM 的读写权限由成员关系、好友关系、黑名单、群状态、父群状态等业务事实决定。
- **Bot 权限**：User Bot 和 App Bot 分流。User Bot 可在它作为成员的群/子区内发消息；App Bot 被设计为 DM-only，Space App Bot 还要求对端仍是绑定 Space 成员。
- **User API Key 权限**：`uk_*` key 会把 uid、client_id、space_id 写入上下文，后续复用的人类 handler 必须通过 authtree 的 TenantScope 或路由 guard 证明不会跨 Space。
- **OBO 权限**：OBO 只授权“以某人身份发言”的替换，不改变 Bot 的租户归属，也不能跳过 Bot 对 channel 的基础可达性；OBO 自身还要检查 grant 与 channel scope。
- **Agent 绑定权限**：Agent 通过 `agent_ref` 占用 User Bot，DB CAS 保证同一 Bot 同时只有一个占用方，解绑也要求同一 `agent_ref`。

## 4. 常见问题

### Q1：octo-server 的 org / 管理面 RBAC 管什么？

管理面仍以登录态为前提，再用系统角色判断是否可进对应管理面。`dashboardReader`、`marketAdmin` 是 octo-server 本地固定角色，不被 octo-lib 的通用 `CheckLoginRole` 直接当作全局 admin；`marketAdmin` 只允许 marketplace 相关管理面，`dashboardReader` 只允许 dashboard 读权限。

来源: `pkg/auth/manager_roles.go#L5-L20`  
来源: `pkg/auth/manager_roles.go#L63-L89`

### Q2：Space 权限管什么？

Space 权限用于租户边界和 Space 管理动作。例如 Space App Bot 管理要求登录用户在目标 Space 中 `role >= admin`；平台路由只能管理 platform bot，Space 路由只能管理该 Space 的 bot，避免跨租户 IDOR。

来源: `modules/app_bot/app_bot.go#L116-L183`  
来源: `modules/app_bot/app_bot.go#L309-L325`

### Q3：频道 ACL 管什么？

频道权限负责“谁能读写哪个 DM/群/子区”。群管理 API 挂登录态中间件；部分群操作要求创建者角色；群解散会同步 WuKongIM disband 信息。子区创建要求创建者是父群活跃成员、父群未解散，再写 thread/thread_member，并创建 IM CommunityTopic channel。

来源: `modules/group/api.go#L92-L144`  
来源: `modules/group/api.go#L221-L223`  
来源: `modules/thread/service.go#L180-L205`  
来源: `modules/thread/service.go#L250-L292`

### Q4：Bot / Agent 身份门禁管什么？

Bot API 根据 Bearer token 前缀分流：`app_` 走 App Bot，其他/`bf_` 走 User Bot；App Bot 必须 published/status=1。Agent 对 User Bot 的占用通过 `agent_ref` 与 DB CAS 实现并发互斥，解绑也要求同一占用方。

来源: `modules/bot_api/auth.go#L10-L42`  
来源: `modules/bot_api/auth.go#L64-L97`  
来源: `modules/botfather/api_user.go#L581-L647`  
来源: `modules/botfather/db.go#L626-L660`

### Q5：User API Key 的权限边界是什么？

`uk_*` API Key 中间件会校验 token、client 状态，并把 `uid/api_key_uid/api_key_space_id/client_id` 写入上下文。authtree 要求每条复用路由声明 TenantScope：有些路由通过 BoundSpaceContext 把 key 绑定的 Space 注入到 handler，有些必须用 route guard 显式限制目标。

来源: `modules/botfather/api_user.go#L36-L75`  
来源: `pkg/authtree/authtree.go#L1-L9`  
来源: `pkg/authtree/authtree.go#L166-L210`  
来源: `pkg/authtree/authtree.go#L216-L260`

### Q6：App Bot 为什么不能进群 / 子区？

App Bot 的发送规则明确只支持 DM，非 Person channel 直接拒绝；scope=space 的 App Bot 还要求 DM 对端仍在绑定 Space 中。bot 树上复用的消息读取路由也加了 `appBotScopeGuard`：DM 读校验对端 Space 成员，群/子区读对 App Bot 一律拒绝。

来源: `modules/bot_api/send.go#L468-L501`  
来源: `modules/bot_api/authtree_guard.go#L17-L45`  
来源: `modules/bot_api/authtree_guard.go#L70-L99`

### Q7：OBO 能绕过 Bot 原本权限吗？

不能。`sendMessage` 先跑 Bot 对 channel 的基础权限检查，再做 OBO grant/scope 检查；源码注释明确 OBO 不能绕过 Bot 原本不具备的 channel 可达性。OBO 只替换 fromUID，不改变租户隔离边界，`space_id` 仍基于 bot 归属注入。

来源: `modules/bot_api/send.go#L166-L244`  
来源: `modules/bot_api/send.go#L252-L260`

### Q8：权限不足时如何处理？

关键鉴权路径普遍采取 fail-closed：查不到角色/成员/Space、scope=space 但缺少 space_id、群已解散、App Bot 未发布等都会拒绝，而不是放行。部分读接口会返回 not found，避免泄露“对象存在但你不可见”。

来源: `modules/app_bot/app_bot.go#L1031-L1083`  
来源: `modules/bot_api/send.go#L503-L522`  
来源: `modules/bot_api/send.go#L538-L587`  
来源: `modules/bot_api/authtree_guard.go#L42-L45`

## 5. 边界易错点

1. **不要说 RBAC 是唯一权限模型。** Space membership、群成员、好友关系、Bot kind、API Key 绑定 Space 都是独立边界。
2. **不要把登录态等同于 Space 管理员。** Space App Bot 管理要 `space_member.role >= admin`。
3. **不要让 App Bot 读写群/子区。** App Bot 策略是 DM-only。
4. **不要把 OBO 理解成超级权限。** OBO 仍受 Bot channel 可达性、grant、scope 与 grantor 当前可读权限约束。
5. **不要把 `uk_*` 复用路由当成人类 session 的简单复制。** authtree 必须说明每条路由的租户锚点。

## 6. 不确定部分

- 通用 `CheckLoginRole` 与 `AuthMiddleware` 的完整实现来自 `octo-lib`，当前知识库只引用 octo-server 侧接入与补强逻辑；如考官追问 octo-lib 内部角色枚举，需要补 octo-lib 源码证据。
- 更完整的频道 ACL 还应补充 message 模块的单条消息读取、群消息读取、黑名单/白名单细节。
