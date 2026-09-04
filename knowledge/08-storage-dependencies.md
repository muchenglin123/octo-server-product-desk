# 08 存储与外部依赖

## 1. 考官可能问什么

1. octo-server 依赖哪些外部服务？
2. MySQL / Redis / 对象存储分别负责什么？
3. 数据表是怎么创建或迁移的？
4. Redis 只是缓存吗，还是也用于 session / 分布式协调？
5. 文件上传支持哪些对象存储后端？
6. 没有 WuKongIM、MySQL、Redis、对象存储时能不能直接跑？
7. 外部依赖缺失时系统如何表现？

## 2. 一句话结论

octo-server 至少依赖 MySQL、Redis、WuKongIM 和文件对象存储；MySQL 是业务权威数据与 migration 目标，Redis 用于 token/session、缓存、限流、OIDC state、跨副本 registry 等，WuKongIM 是消息投递和 IM 同步核心，对象存储承载文件/图片/附件上传下载。部分依赖在启动期 fail-fast（MySQL ping、session Redis Lua probe、migration），部分模块在配置缺失时注册但请求失败或降级。

## 3. 产品解释

- **MySQL**：保存用户、Space、群、消息扩展、Bot、App Bot、配置、迁移记录等业务状态。启动时连接 MySQL 并执行各模块 SQL migration。
- **Redis**：承载会话 token、缓存、限流桶、OIDC state/bind store、App Bot auth registry、后台任务/队列等场景。Redis TLS 配置由统一 builder 处理。
- **WuKongIM**：负责 IM channel、消息投递、CMD、最近会话、消息同步、清未读等底层能力；octo-server 是控制面和业务编排层。
- **对象存储**：文件模块支持 MinIO、Aliyun OSS、Tencent COS、Qiniu、AWS S3、SeaweedFS 等后端；不同后端对预签名上传/下载支持不同。
- **外部服务**：语音 adapter 依赖 `SPEECH_SERVICE_URL`；OIDC/OAuth 依赖上游 IdP；internal resolve 依赖内部 token；Docker/OOTB 部署还包含 admin/web/matter/summary/nginx 等旁路组件。

## 4. 常见问题

### Q1：MySQL 主要存储什么？

MySQL 是 octo-server 的业务数据库。代码中大量模块通过 `ctx.DB()` 读写用户、Space、群、Bot、App Bot、配置等表；例如 App Bot 管理查询 `space_member` 判断 Space 管理权限，BotFather 创建 Bot 时写 `space_member`、friend、robot 等关系。

来源: `modules/app_bot/app_bot.go#L156-L183`  
来源: `modules/botfather/api_user.go#L209-L284`  
来源: `modules/common/db_app_config.go#L23-L56`

### Q2：数据库表如何创建和迁移？

MySQL 连接由 `pkg/db.NewMySQL` 创建，启动时会 ping 数据库；如果开启 migration，会用 `sql-migrate` 执行迁移。迁移源会递归扫描 SQL 文件并按 migration ID 排序。主程序在 `module.Setup(ctx)` 前先执行 legacy migration ID rewrite 和 thread schema reconciliation，避免历史库升级时触发 sql-migrate 安全检查或重复建表。

来源: `pkg/db/mysql.go#L15-L45`  
来源: `pkg/db/mysql.go#L58-L109`  
来源: `main.go#L454-L486`  
来源: `pkg/db/migrate_compat.go#L26-L44`

### Q3：Redis 在系统中负责什么？

Redis 不只是普通缓存。启动时认证 session store 会 probe Redis Lua 支持，失败直接 panic；Redis 也用于 rate limit、session、card action dispatch 等指标注册。octo-server 自己创建 Redis client 时通过统一 options builder 应用 TLS、密码与 instrumentation。

来源: `main.go#L214-L227`  
来源: `main.go#L440-L447`  
来源: `pkg/db/redis.go#L8-L17`  
来源: `pkg/redis/options.go#L15-L69`

### Q4：对象存储用于什么场景？

文件模块提供认证后的文件预览、上传地址、上传、预签名上传、预签名下载接口，也把预签名能力开放给 `uk_*` User API Key 树。它会根据 `FileService` 选择 MinIO、OSS、Qiniu、COS、S3、SeaweedFS 等实现。

来源: `modules/file/api.go#L83-L130`  
来源: `modules/file/service.go#L32-L56`  
来源: `modules/file/service.go#L58-L100`

### Q5：预签名上传/下载有什么依赖要求？

模板说明浏览器直传要求对象存储 bucket 配好 CORS；SigV4/OSS 签名覆盖 content-type、content-disposition 等 header，浏览器必须原样带上，否则会被对象存储拒绝。不同后端 presigned PUT/GET 支持不同：MinIO/COS/OSS 支持 PUT+GET，Qiniu 只支持 signed GET，SeaweedFS 不支持 presign primitive。

来源: `configs/tsdd.yaml#L69-L103`  
来源: `modules/file/service.go#L45-L55`  
来源: `modules/file/service.go#L147-L160`

### Q6：文件安全边界是否完全由 Space 控制？

不是。authtree 注释明确指出文件下载 URL 路由是 `ScopeUnscoped`，对象保密今天不在该路由层强制，主要依赖 object key 高熵不可枚举、消息读路由 Space-confined 等；真正收口需要归属表、capability token 或 bucket policy 等更底层改造。

来源: `pkg/authtree/authtree.go#L74-L84`  
来源: `pkg/authtree/authtree.go#L103-L130`  
来源: `modules/file/api.go#L101-L130`

### Q7：WuKongIM 依赖在哪里体现？

配置模板有 `wukongIM.apiURL` 与 `managerToken`。运行中 group/thread/message/bot_api 等会调用 WuKongIM 能力：例如 group 解散推送 disband flag，thread 创建 CommunityTopic channel，Bot 消息发送最终交给 IM 发送。

来源: `configs/tsdd.yaml#L21-L24`  
来源: `modules/group/api.go#L385-L407`  
来源: `modules/thread/service.go#L250-L292`  
来源: `modules/bot_api/send.go#L172-L178`

### Q8：外部服务缺失时系统如何表现？

MySQL 连接失败或 ping 失败会 panic；session Redis Lua probe 失败会 panic；migration 兼容处理失败会 panic。语音 adapter 缺 `SPEECH_SERVICE_URL` 时启动只 warning，但请求会失败。对象存储配置错误通常在具体上传/签名请求触发错误。

来源: `pkg/db/mysql.go#L18-L35`  
来源: `main.go#L214-L227`  
来源: `main.go#L454-L486`  
来源: `modules/voice_adapter/1module.go#L14-L23`

## 5. 边界易错点

1. **不要说 Redis 只是缓存。** 它也承载 session、Lua 原子写、限流、OIDC state、跨副本状态等。
2. **不要说对象存储下载天然按 Space 隔离。** 当前 `/file/download/url` 明确是未完全收口的边界。
3. **不要把 WuKongIM 说成可选。** 本地 Go 运行需要可达 WuKongIM、MySQL、Redis；OOTB stack 会一起启动这些服务。
4. **不要说所有对象存储都支持预签名 PUT。** 支持矩阵在模板里写得很明确。
5. **不要把 migration 顺序理解为模块 import 顺序。** migration ID/时间戳排序才是关键。

## 6. 不确定部分

- 各张业务表的完整字段与迁移历史还需要进一步从 `modules/*/sql` 做表级索引。
- WuKongIM 的服务端内部存储不在 octo-server 仓库内；本卡只说明 octo-server 如何依赖和调用 IM 控制面。
