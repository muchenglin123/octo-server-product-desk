# octo-server 知识卡片：API 与错误约定

## 1. 考官可能问什么

1. octo-server 的错误响应体长什么样？是否兼容老客户端？
2. 错误码在哪里注册？命名规则是什么？
3. i18n 文案从哪里来？缺翻译时怎么兜底？
4. HTTP 状态码到底看响应头，还是看 body 里的 `error.http_status`？
5. `ResponseErrorL` 和 `ResponseErrorLWithStatus` 有什么区别？
6. `details` 会不会泄露 token、uid、raw_err？
7. 已迁移模块和未迁移模块怎么区分？

## 2. 一句话结论

octo-server 正在使用一套基于 `wkhttp.ErrorRenderer` 的统一错误响应机制：业务层通过 `httperr.ResponseErrorL*` 传入注册过的 i18n 错误码，最终输出新版 `error.{code,message,details,http_status}` 与老版 `msg/status` 双信封；但全仓库不是 100% 完成迁移，部分老模块仍存在旧式 `c.ResponseError`。

## 3. 产品解释

- 新客户端读取 `error.code`、`error.message`、`error.details`、`error.http_status`。
- 老客户端仍可读取 `msg`、`status`。
- 默认 `ResponseErrorL` 为兼容旧客户端，wire HTTP status 仍可能是 400；`ResponseErrorLWithStatus` 才把 transport status 设为 canonical HTTPStatus。
- 错误码必须集中注册，不能临时拼。
- `details` 只允许白名单字段，内部错误不会原样透出。

## 4. 源码证据

1. `ResponseErrorL` 是业务侧本地化错误入口，并说明会保留 legacy HTTP/body `status=400` 兼容路径。  
   来源: `pkg/httperr/respond.go#L13-L22`

2. `ResponseErrorLWithStatus` 用于保留真实 HTTP transport status 的接口；注释明确 body envelope 与 `ResponseErrorL` 一致。  
   来源: `pkg/httperr/respond.go#L26-L49`

3. `respondL` 根据 `useSemanticStatus` 决定 wire status：默认 400；WithStatus 时使用注册错误码的 `HTTPStatus`。  
   来源: `pkg/httperr/respond.go#L53-L80`

4. i18n renderer 明确输出双信封：新版 `error.{code,message,details,http_status}`，老版 `msg/status`。  
   来源: `pkg/i18n/renderer.go#L27-L30`

5. renderer 实际写出的 JSON 结构包含 `error.code`、`error.message`、`error.details`、`error.http_status`，并同时包含 legacy `msg` 和 `status`。  
   来源: `pkg/i18n/renderer.go#L53-L65`

6. `Internal=true` 时，renderer 不使用业务错误的默认文案，而是统一翻译 `err.shared.internal`，避免内部错误泄露。  
   来源: `pkg/i18n/renderer.go#L68-L81`

7. renderer 会再次过滤 details，即使绕过 `httperr.ResponseErrorL` 直接调用 `RenderError`，也只按注册错误码白名单透传。  
   来源: `pkg/i18n/renderer.go#L84-L97`

8. 错误码注册表要求所有 user-visible 错误码通过 `Register` 登记；ID 全局唯一，重复注册会 panic。  
   来源: `pkg/i18n/codes/registry.go#L1-L18`

9. 错误码 ID 只能是 `err.shared.*` 或 `err.server.*`，segment 只能小写字母、数字、下划线。  
   来源: `pkg/i18n/codes/registry.go#L28-L43`

10. `Code` 结构定义了 `ID`、`HTTPStatus`、`DefaultMessage`、`DefaultMessages`、`SafeDetailKeys`、`Internal`。  
    来源: `pkg/i18n/codes/registry.go#L45-L66`

11. shared 错误码集中注册了通用鉴权、限流、参数、not_found、internal 等错误，并给 `rate.limited` 设置 429、给 `internal` 设置 500。  
    来源: `pkg/i18n/codes/shared.go#L5-L20`  
    来源: `pkg/i18n/codes/shared.go#L67-L105`

12. i18n bundle 会把 `codes.All()` 中的 `DefaultMessage` 注入为 source 语言消息，保证 fallback。  
    来源: `pkg/i18n/bundle.go#L35-L69`

13. Localizer 的 fallback 链路明确：requested lang → fallback lang → source lang → `DefaultMessages` → `DefaultMessage` → code ID。  
    来源: `pkg/i18n/localizer.go#L9-L24`  
    来源: `pkg/i18n/localizer.go#L69-L94`

14. 语言协商优先级为 trusted `X-Octo-Lang`、URL `lang`、cookie `i18n_lang`、user.language、`Accept-Language`、默认语言。  
    来源: `pkg/i18n/lang.go#L12-L18`  
    来源: `pkg/i18n/lang.go#L50-L82`

15. i18n 中间件会在请求早期协商语言，写入 request context，设置 `Content-Language`，并追加 `Vary`。  
    来源: `pkg/i18n/middleware.go#L16-L34`

## 5. 边界与易错点

- `error.http_status` 是 body 中的语义状态；wire status 在兼容接口里可能仍是 400。
- 不是所有模块都完全迁到新错误体系，看到旧 `ResponseError` 不能误判为全局规范。
- `details` 不能随便透传调试信息，只能透传注册错误码允许的字段。

## 6. 考试可直接回答

> 结论：octo-server 的新错误约定是 i18n 双信封：新版客户端读 `error.code/message/details/http_status`，老客户端继续读 `msg/status`。业务层通过 `httperr.ResponseErrorL*` 调用 wkhttp renderer。默认接口为兼容旧客户端 wire status 可能还是 400；需要真实 HTTP 状态时使用 `ResponseErrorLWithStatus`。

## 7. 待补 / 不确定项

- 若考官追问某个具体模块是否已完全迁移，需要针对该模块逐文件查旧 `ResponseError` 残留。
