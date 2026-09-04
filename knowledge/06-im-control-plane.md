# octo-server 知识卡片：IM 控制面

## 1. 考官可能问什么

1. octo-server 和 WuKongIM 的职责边界是什么？
2. channel、group、thread、message、conversation_ext 分别管什么？
3. 发消息到底是走 octo-server，还是直连 WuKongIM？
4. DM / 群 / 子区的 channel_id 和 channel_type 怎么表达？
5. 创建子区时，octo-server 做了哪些事，WuKongIM 做了哪些事？
6. 会话列表、关注 Tab、最近 Tab 是 IM 直接返回的吗？

## 2. 一句话结论

octo-server 是 IM 产品控制面和业务编排层，负责身份鉴权、权限校验、群/子区/关注关系等业务状态、Space 隔离和 payload 注入；WuKongIM 是消息投递、频道同步、最近会话、搜索、清未读等底层 IM 能力。

## 3. 产品解释

- **group**：群的业务资料和成员关系在 octo-server 的 DB 里维护。
- **thread**：子区是 octo-server 的业务对象，但在 IM 层表现为 `ChannelTypeCommunityTopic` 的频道，`channel_id = groupNo + "____" + shortID`。
- **message**：发消息入口通常先走 octo-server 做鉴权、Space/payload/mention 等处理，最终调用 WuKongIM 发送。
- **conversation_ext**：不是 IM 消息表，而是用户侧关注/取关/排序/子区物化的 server 侧扩展状态。

## 4. 源码证据

1. BotAPI 是 bot-facing gateway，集中挂 `/v1/bot/*`，并持有 group/thread/message 相关服务。  
   来源: `modules/bot_api/bot_api.go#L34-L43`

2. Bot 消息发送最终通过 `dispatchMsgSendReq` 进入 `ctx.SendMessageWithResult`，即交给 IM 发送。  
   来源: `modules/bot_api/bot_api.go#L207-L214`

3. Bot API 路由里挂载了群信息、群成员、GROUP.md、建群、改群、加/移成员、thread API、消息编辑等 server 端接口。  
   来源: `modules/bot_api/bot_api.go#L423-L453`

4. DM 的 channel 解析当前是 no-op，注释说明 WuKongIM 对 DM 使用裸 uid，不加 Space 前缀。  
   来源: `modules/bot_api/bot_api.go#L502-L510`

5. `/v1/bot/sendMessage` 请求体包括 `channel_id`、`channel_type`、`on_behalf_of`、`payload`；server 先校验必填字段。  
   来源: `modules/bot_api/send.go#L36-L49`  
   来源: `modules/bot_api/send.go#L53-L71`

6. Bot 发消息先做权限检查，再解析 channel；OBO 情况还会先校验 grant/scope，之后才替换 `fromUID`。  
   来源: `modules/bot_api/send.go#L172-L180`  
   来源: `modules/bot_api/send.go#L227-L244`

7. DM 的 `payload.space_id` 由 server 权威注入；注释明确说 WuKongIM 对 DM 仅按裸 uid 路由、无 Space 概念。  
   来源: `modules/bot_api/send.go#L252-L260`

8. Bot 消息最终构造 `config.MsgSendReq`，包含 `ChannelID`、`ChannelType`、`FromUID`、`Payload`，随后调用发送分发。  
   来源: `modules/bot_api/send.go#L426-L436`

9. App Bot 发送权限是 DM-only：非 Person channel 直接拒绝；DM 还要检查好友关系和 Space 成员关系。  
   来源: `modules/bot_api/send.go#L468-L501`

10. User Bot 发群消息时，server 会先查群状态，避免解散群中 WuKongIM 返回 200 但实际拒收导致 bot 误以为发送成功。  
    来源: `modules/bot_api/send.go#L503-L522`

11. thread 模块启用后注册 API、Swagger、SQL 和 IM datasource，处理 `ChannelTypeCommunityTopic`。  
    来源: `modules/thread/1module.go#L45-L73`  
    来源: `modules/thread/1module.go#L74-L96`

12. message 模块组合注册 message、conversation、manager、sidebar，并注入 thread/group/conversation_ext 的鉴权与枚举器。  
    来源: `modules/message/1module.go#L26-L57`  
    来源: `modules/message/1module.go#L59-L103`

13. conversation_ext 模块注册服务/SQL 和 `/v1/follow` 路由。  
    来源: `modules/conversation_ext/1module.go#L73-L105`

## 5. 边界与易错点

- 不要把 octo-server 当成消息内核；真正消息投递依赖 WuKongIM。
- 不要把 thread 当普通 group；它在 IM 层是 CommunityTopic。
- DM 使用裸 uid，Space 信息通过可信 payload 注入，不是 WuKongIM 原生理解 Space。
- App Bot 群能力受限，不能简单套用 User Bot 群权限。

## 6. 考试可直接回答

> 结论：octo-server 是 IM 控制面，不是消息内核。它负责鉴权、群/子区业务状态、Space 隔离、payload 注入和 Bot API 编排；真正的发送、同步、最近会话、搜索、清未读等底层能力由 WuKongIM 提供。比如 Bot 发消息会先走 octo-server 做权限校验和 payload 处理，最后才构造 `MsgSendReq` 调 IM 发送。

## 7. 待补 / 不确定项

- 若考官追问 WuKongIM 内核实现，需要查 WuKongIM 仓库；当前卡片只覆盖 octo-server 控制面源码。
