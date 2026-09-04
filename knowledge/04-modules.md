# octo-server 知识卡片：业务模块清单

## 1. 考官可能问什么

1. octo-server 的 `modules/` 下到底有多少个业务模块？
2. 哪些模块会通过 `internal/modules.go` 被主程序 blank import 注册？
3. `modules/` 目录存在但没有被 blank import 的包算不算启用模块？
4. module 的注册机制是什么？
5. README 架构表和实际目录不一致时，以哪个为准？
6. thread 模块为什么说 schema 总注册，但 API/worker 可被开关禁用？
7. Bot 相关模块有哪些，各自负责什么？
8. IM 控制相关模块有哪些？

## 2. 一句话结论

`octo-server/modules` 当前有 41 个一级目录，但主程序通过 `internal/modules.go` blank import 了其中 38 个包；`botidentity`、`cardtrust`、`source` 是存在于 `modules/` 下的 Go 包/支撑包，但不在主模块 blank import 列表中，不能简单等同为运行时注册模块。

## 3. 产品解释

从产品角度看，`octo-server` 不是一个单体接口文件，而是把用户、群、消息、Bot、OIDC、文件、通知、工作台等能力拆成模块。每个启用模块通常在自己的 `1module.go` 里调用 `register.AddModule`，声明模块名、API 路由、SQL migration、Swagger、服务生命周期或 IM datasource。主程序通过 `internal/modules.go` 的 blank import 触发这些模块的 `init()` 注册。

## 4. 源码证据

1. **模块 import 顺序不是 migration 的真实执行顺序；SQL migration 由文件时间戳全局排序，Go init 顺序由包依赖图决定。** 这意味着不能把 `internal/modules.go` 的排列顺序误解为绝对加载/迁移顺序。  
   来源: `internal/modules.go#L1-L18`

2. **主模块通过 blank import 引入模块包，触发各模块 init 注册。**  
   来源: `internal/modules.go#L22-L78`

3. **`internal/modules.go` 中记录了 runtime 模块已移除，runtime/bot orchestration 由单独的 octo-fleet 服务负责。**  
   来源: `internal/modules.go#L53-L57`

4. **`usersecret` 模块在主模块中被引入，注释说明它提供用户外部密钥别名表与 resolve 能力，并通过 bf_ bot token 反查 owner。**  
   来源: `internal/modules.go#L63-L66`

5. **`bot_api` 在 `app_bot` 之前引入，注释说明 app_bot 与 bot_api 有运行期和 Go 包级依赖；但真正 init 顺序仍由依赖图决定。**  
   来源: `internal/modules.go#L67-L72`

6. **最基础的模块注册模式是 `register.AddModule` 返回 `register.Module`，包含 `SetupAPI` 与 `SQLDir`。**  
   来源: `modules/base/1module.go#L14-L24`

7. **user 模块一次注册了用户模块和 friend 模块，说明一个目录可以注册多个逻辑模块。**  
   来源: `modules/user/1module.go#L26-L39`  
   来源: `modules/user/1module.go#L104-L114`

8. **group 模块注册了 HTTP API、SQL、Swagger，同时向 user 模块反向注入群成员检查、外部成员字段、共同群检查等能力。**  
   来源: `modules/group/1module.go#L22-L49`

9. **group 模块还是 IM datasource：向 WuKongIM 提供群 channel info、订阅者、黑名单、白名单等控制面数据。**  
   来源: `modules/group/1module.go#L50-L105`

10. **thread 模块即使运行时关闭，也仍注册 SQL schema；只有 API 与 archive worker 受 `DM_THREAD_ON` 控制。**  
    来源: `modules/thread/1module.go#L24-L43`

11. **thread 模块启用后注册 Start/Stop、API、Swagger、SQL 和 IM datasource，处理 ChannelTypeCommunityTopic。**  
    来源: `modules/thread/1module.go#L45-L73`  
    来源: `modules/thread/1module.go#L74-L96`

12. **message 模块不仅注册 message API，还注册 conversation、manager、sidebar，并在这里组合注入 thread/group/conversation_ext 的鉴权与枚举器，避免模块循环依赖。**  
    来源: `modules/message/1module.go#L26-L57`  
    来源: `modules/message/1module.go#L59-L103`

13. **conversation_ext 模块分两次注册：一次注册服务/SQL 单例，一次注册 `/v1/follow` 路由。**  
    来源: `modules/conversation_ext/1module.go#L73-L87`  
    来源: `modules/conversation_ext/1module.go#L89-L105`

14. **space 模块也在一个目录下注册 `space` 和 `space_manager` 两个逻辑模块，并用 sync.Once 共享同一个 Space 实例。**  
    来源: `modules/space/1module.go#L17-L42`  
    来源: `modules/space/1module.go#L44-L52`

15. **Bot 相关模块各自注册：robot、botfather、bot_api、app_bot、bot_provision，分别覆盖机器人基础资料、用户 Bot 管理、Bot API、应用 Bot、跨服务 bot provision。**  
    来源: `modules/robot/1module.go#L16-L28`  
    来源: `modules/botfather/1module.go#L16-L26`  
    来源: `modules/bot_api/1module.go#L13-L22`  
    来源: `modules/app_bot/1module.go#L13-L22`  
    来源: `modules/bot_provision/1module.go#L8-L16`

16. **`incomingwebhook` 模块通过 BussDataSource 让 iwh_ 合成身份可被 channel/user 查询解析为发送者名和头像。**  
    来源: `modules/incomingwebhook/1module.go#L13-L32`

17. **`webhook` 和 `notify` 模块都有 Start/Stop 生命周期，说明部分模块不仅注册 API/SQL，也启动后台消费者或 worker。**  
    来源: `modules/webhook/1module.go#L13-L30`  
    来源: `modules/notify/1module.go#L13-L31`

18. **`messages_search` 注释说明 web / bot / uk 三条路由树共用 Shared Handler，这也是跨入口复用业务能力的例子。**  
    来源: `modules/messages_search/1module.go#L13-L24`

19. **`internal_resolve` 是内部服务接口模块，注释说明它服务缺少用户 token 的内部消费者，并用不同 X-Internal-Token 做 fail-closed 认证。**  
    来源: `modules/internal_resolve/1module.go#L1-L8`  
    来源: `modules/internal_resolve/1module.go#L15-L23`

20. **`agentmailgateway` 模块注册 Agent mail 代理 API，是 Agent 相关但不属于核心 Bot API 的边界能力。**  
    来源: `modules/agentmailgateway/1module.go#L8-L16`

## 5. 41 个目录与启用情况

### modules/ 下 41 个一级目录

`agentmailgateway`、`app_bot`、`backup`、`base`、`bot_api`、`bot_mention`、`bot_provision`、`botfather`、`botidentity`、`card_template_catalog`、`cardtrust`、`category`、`channel`、`common`、`conversation_ext`、`file`、`group`、`incomingwebhook`、`integration`、`internal_resolve`、`message`、`messages_search`、`notification`、`notify`、`oidc`、`opanalytics`、`openapi`、`qrcode`、`report`、`robot`、`search`、`source`、`space`、`statistics`、`sticker`、`thread`、`user`、`usersecret`、`voice_adapter`、`webhook`、`workplace`。

### 被 `internal/modules.go` blank import 的 38 个目录

除 `botidentity`、`cardtrust`、`source` 外，其余 38 个目录都在 `internal/modules.go` 中被 blank import。

### 目录存在但不在 blank import 列表中

- `botidentity`：Bot 身份解析支撑包，被其他模块直接 import 使用，不靠 blank import 注册 HTTP 模块。
- `cardtrust`：卡片可信处理支撑包。
- `source`：source 相关支撑包。

## 6. 边界与易错点

1. **不要直接说 “41 个模块全部启用”。** 更准确是：`modules/` 有 41 个一级目录，其中 38 个被主模块 blank import；有些目录是支撑包而非运行时注册模块。
2. **不要把 import 顺序当 migration 顺序。** 注释明确 migration 按 SQL 文件时间戳排序。
3. **一个目录可以注册多个逻辑模块。** 例如 user 注册 user/friend，message 注册 message/conversation/manager/sidebar/thread auth，space 注册 space/space_manager。
4. **thread 是典型 feature gate。** schema 总注册，但 API 和 archive worker 受环境变量控制。
5. **有些模块不只是 API。** group/thread 给 IM 提供 datasource，notify/webhook 有生命周期 worker。

## 7. 考试可直接回答

> 结论：`octo-server/modules` 目录下有 41 个一级目录，但不能粗暴说 41 个都以同样方式启用。主程序通过 `internal/modules.go` blank import 了 38 个模块包，触发它们的 `init()` 调用 `register.AddModule` 完成 API、SQL、Swagger、服务生命周期或 IM datasource 注册。`botidentity`、`cardtrust`、`source` 虽在 modules 下，但不在 blank import 列表里，更像被其他模块引用的支撑包。另一个易错点是 import 顺序不是 migration 执行顺序，源码注释明确 migration 由 SQL 文件时间戳全局排序。

## 8. 待补 / 不确定项

- 还需要继续为每个模块补一行产品职责说明，形成完整模块字典。
- `cardtrust` 与 `source` 的具体产品职责需要结合调用方继续确认。
- 哪些模块在特定配置下不可用，需要后续结合配置文件和环境变量开关逐项确认。
