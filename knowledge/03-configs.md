# 03 配置

## 1. 考官可能问什么

1. octo-server 默认读取哪个配置文件？能用环境变量覆盖吗？
2. `configs/tsdd.yaml` 里主要有哪些配置段？
3. 哪些配置缺失或错误会导致启动失败？
4. Redis TLS、token 过期时间、release 模式万能验证码这些配置如何校验？
5. OIDC、内部 token、语音服务这类模块配置从哪里来？
6. 配置项和业务模块有什么关系？

## 2. 一句话结论

octo-server 启动默认读取 `configs/tsdd.yaml`，同时用 Viper 支持 `TS_*` 环境变量覆盖 YAML 字段；部分模块还直接读取 `DM_*` / `OCTO_*` / `SPEECH_SERVICE_URL` 等环境变量。启动期会对关键配置做 fail-fast 校验，例如配置文件不可读、token TTL 非法、release 模式配置万能验证码、认证 session Redis 不支持 Lua、i18n locale 非法、迁移兼容处理失败等都会导致 panic 或模块禁用。

## 3. 产品解释

配置分三层：

1. **核心 YAML / TS 环境变量层**：`mode`、监听地址、WuKongIM、MySQL、Redis、外网地址、对象存储、推送、缓存等，通常在 `configs/tsdd.yaml` 中定义，并可通过 `TS_` 前缀环境变量覆盖。
2. **模块自管环境变量层**：OIDC、thread、内部 resolve、card action、语音 adapter 等因为不完全在 octo-lib Config 结构体中，直接读取 `DM_*`、`OCTO_*`、`SPEECH_SERVICE_URL` 等环境变量。
3. **DB 动态配置层**：例如 `app_config`、system settings、file policy 等部分运行时配置存数据库，模块启动时挂载配置源或读取快照。

## 4. 常见问题

### Q1：默认配置文件在哪里，如何读取？

主程序通过 `-config` flag 指定配置文件，默认值是 `configs/tsdd.yaml`；`loadConfigFromFile` 用 Viper 读取文件，读取失败直接 panic。随后设置 env prefix 为 `TS`，把配置 key 里的 `.` 替换成 `_`，并启用 `AutomaticEnv()`，所以例如 `db.redisAddr` 可由 `TS_DB_REDISADDR` 覆盖。

来源: `main.go#L98-L156`

### Q2：`configs/tsdd.yaml` 主要有哪些配置段？

模板包含基础运行配置、Webhook HMAC 密钥、WuKongIM API/manager token、MySQL/Redis、外网地址、文件服务与对象存储、管理员 UID、头像、短号、机器人、第三方登录、cache 等段落。

来源: `configs/tsdd.yaml#L1-L40`  
来源: `configs/tsdd.yaml#L65-L156`  
来源: `configs/tsdd.yaml#L218-L259`

### Q3：哪些配置错误会导致启动失败？

配置文件读取失败会 panic；`cache.tokenExpire` 会在 Viper 读取并应用 env override 后校验，非法 duration、非正数、小于 1ms、超过 720h 都会报错；release 模式下配置 `smsCode` 会被拒绝，避免万能验证码后门。

来源: `main.go#L98-L156`  
来源: `internal/tokenlifecycle/config.go#L12-L52`  
来源: `modules/base/common/testcode.go#L41-L48`

### Q4：Redis 配置如何处理 TLS？

`configs/tsdd.yaml` 提供 `redisAddr`、`redisPass`、`redisTLS`、`redisTLSInsecureSkipVerify`、`redisTLSCAFile`。octo-server 自己创建裸 Redis client 时应使用 `pkg/redis.BuildOptions` / `NewInstrumentedClient`，统一应用 Addr、Password、TLSConfig；TLS CA 文件解析失败属于启动期配置错误，`MustBuildOptions` 会 panic。

来源: `configs/tsdd.yaml#L26-L34`  
来源: `pkg/redis/options.go#L15-L44`  
来源: `pkg/redis/options.go#L47-L69`

### Q5：OIDC 配置为什么不在 YAML 里？

OIDC 模块注释明确：`TS_*` 是 Viper 管理的核心配置；`DM_*` 是模块自管功能开关/第三方对接配置。OIDC 因 octo-lib 暂未支持 OIDC 配置块，所以目前走环境变量，例如 `DM_OIDC_ENABLED`、`OCTO_OIDC_EXCHANGE_ENABLED`、`DM_OIDC_PROVIDER_*`。

来源: `modules/oidc/config.go#L20-L33`  
来源: `modules/oidc/config.go#L34-L53`  
来源: `modules/oidc/config.go#L129-L153`

### Q6：内部服务 token 配置如何校验？

`internal_resolve` 使用 `OCTO_DRIVE_INTERNAL_TOKEN` 和 `X-Internal-Token`。配置读取会拒绝未设置、长度小于 32 字节、与 sibling fixed internal token 冲突的值；错误信息不包含 token 明文。

来源: `modules/internal_resolve/config.go#L7-L23`  
来源: `modules/internal_resolve/config.go#L40-L54`  
来源: `modules/internal_resolve/config.go#L85-L118`

### Q7：语音服务配置缺失会怎样？

`voice_adapter` 从环境加载配置；如果 `SPEECH_SERVICE_URL` 未设置，会在启动时打 warning，并明确说明 voice adapter 请求会失败，但模块仍注册 API 和 SQL。

来源: `modules/voice_adapter/1module.go#L14-L31`

### Q8：文件服务配置有哪些重点？

文件服务通过 `fileService` 选择 MinIO、Aliyun OSS、Tencent COS、Qiniu、SeaweedFS、S3 等后端。模板中特别说明浏览器直传预签名 PUT 要求对象存储 CORS、SigV4/OSS 签名 header 必须被浏览器原样带上；不同后端对 presigned PUT/GET 的支持不同。

来源: `configs/tsdd.yaml#L69-L135`  
来源: `modules/file/service.go#L58-L100`  
来源: `modules/file/service.go#L139-L160`

### Q9：配置项和业务模块之间有什么关系？

主程序把配置装入 `config.Context` 后交给模块。模块通过 `ctx.GetConfig()` 读取配置；例如 file 模块根据 `FileService` 选择对象存储后端，group 模块读取 system group/avatar 路径等配置，Bot/API/IM 相关逻辑读取外网地址和 Space 配置。

来源: `main.go#L152-L156`  
来源: `modules/file/service.go#L58-L100`  
来源: `modules/group/api.go#L519-L571`

## 5. 边界易错点

1. **不要说所有配置都在 YAML。** OIDC、内部 token、语音 adapter、部分限流和 CORS 等配置走环境变量。
2. **不要把 TS 与 DM/OCTO env 混淆。** `TS_*` 主要覆盖 Viper/YAML；`DM_*`、`OCTO_*` 多为模块自管。
3. **不要把 `cache.tokenExpire: 30d` 当合法 Go duration。** 模板明确用 `720h`，代码用 `time.ParseDuration`。
4. **release 模式不能配置万能验证码。** 这是启动期安全校验。
5. **对象存储预签名上传不是只配 bucket 就够。** 浏览器 CORS 与签名 header 契约也必须满足。

## 6. 不确定部分

- `octo-lib/config.Config` 的完整字段定义不在当前 octo-server 仓库中；本知识卡只基于 octo-server 的读取、校验、模板与模块使用方式。
- 完整部署环境变量矩阵需要结合 `octo-deployment` 仓库补充；当前只覆盖 octo-server 侧源码证据。
