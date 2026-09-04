# 09 构建与发布

## 1. 考官可能问什么

1. octo-server 如何本地构建？
2. go build 的入口和产物是什么？
3. Dockerfile 和 Dockerfile.ghcr 有什么区别？
4. Makefile 里的 build / deploy / push 是否都是推荐发布方式？
5. octo-server 和 octo-deployment 是什么关系？
6. GitHub Actions 如何发布镜像？
7. 不启动 MySQL、Redis、WuKongIM 能不能直接跑？

## 2. 一句话结论

octo-server 是 Go 根包构建出的单二进制服务，本地应使用 `go build -o octo-server .`；Dockerfile 采用多阶段构建并把二进制、assets、configs 放入 Alpine 镜像；Dockerfile.ghcr 则是预构建二进制拷贝进 Debian slim 的运行镜像。完整一键体验/部署不在本仓库，而以 `octo-deployment` 为单一来源；本仓 Makefile 中旧 deploy/push 目标带私有 registry 历史遗留，不应作为 canonical release surface。

## 3. 产品解释

- **本地开发**：适合后端开发者，自己准备 MySQL、Redis、WuKongIM 和可选对象存储，然后 `go build -o octo-server .`，再用 `--config` 指向配置文件运行。
- **单服务镜像**：本仓 Dockerfile 只构建/运行 octo-server 本身，不包含 MySQL、Redis、WuKongIM、web/admin 等全套依赖。
- **完整 OOTB 部署**：应使用 `Mininglamp-OSS/octo-deployment`，里面编排 server + admin + web + matter + smart-summary + WuKongIM + MySQL + Redis + MinIO + nginx。
- **发布流水线**：`.github/workflows/docker-publish.yml` 在 `v*` tag 或手动 dispatch 时发布 Docker Hub 多架构镜像，并有 tag 校验、并发控制、environment gate 与 Buildx 构建。

## 4. 常见问题

### Q1：octo-server 如何本地构建？

Quickstart 明确建议 clone 后执行 `go build -o octo-server .`；`go build ./...` 只编译检查多个包，不会留下可执行二进制。运行时 `--config` flag 必须放在 subcommand 前面，例如 `./octo-server --config /path/to/tsdd.yaml api`。

来源: `QUICKSTART.md#L50-L75`  
来源: `QUICKSTART.md#L103-L116`

### Q2：Go 版本和私有依赖有什么要求？

Quickstart 要求 Go ≥ 1.25。`BUILDING.md` 说明项目依赖 sibling repository `octo-lib`，预发布阶段如果 `go build ./...` 报 missing go.sum entry，需要 clone sibling repo 并在本地 `go.mod` 加 replace 指向 `../octo-lib`。

来源: `QUICKSTART.md#L52-L58`  
来源: `BUILDING.md#L3-L27`

### Q3：Dockerfile 做了什么？

Dockerfile 使用 `golang:1.25` 作为 build stage，下载 Go 依赖后执行 `CGO_ENABLED=0 GOOS=linux go build`，通过 ldflags 注入 commit/date/version/tree state，输出 `/go/release/app`；prod stage 使用 `alpine:3.21`，复制二进制、assets、configs，设置 Asia/Shanghai 时区，入口是 `/home/app`。

来源: `Dockerfile#L11-L32`  
来源: `Dockerfile#L35-L48`

### Q4：Dockerfile.ghcr 和 Dockerfile 有什么区别？

Dockerfile.ghcr 不在镜像内执行 Go 编译，而是假设已有 `linux_${TARGETARCH}` 二进制；它以 `debian:bookworm-slim` 为运行时镜像，安装 ca-certificates/tzdata，复制 assets、configs、对应架构二进制到 `/app/main`，CMD 执行 `/app/main`。

来源: `Dockerfile.ghcr#L1-L17`

### Q5：Makefile 里的目标是否都是推荐发布方式？

`make build` 是 `docker build -t octo-server .`。但 `BUILDING.md` 明确说明 `push/deploy/deploy-v2` 是合并前历史目标，硬编码私有 Aliyun registry，且 `make push` 还有旧 tag 遗留；它们不是 canonical release surface，不应使用。

来源: `Makefile#L1-L13`  
来源: `BUILDING.md#L43-L62`

### Q6：octo-server 和 octo-deployment 是什么关系？

README、Quickstart、BUILDING 都说明：一键 Docker Compose 全栈部署使用 `Mininglamp-OSS/octo-deployment`，它是 OOTB 部署单一来源；本仓以前的 compose stack 已移除。

来源: `README.md#L45-L66`  
来源: `QUICKSTART.md#L19-L46`  
来源: `BUILDING.md#L33-L55`

### Q7：GitHub Actions 如何发布镜像？

`docker-publish.yml` 在 `v*` tag push 或 workflow_dispatch 触发。它先做无 secret 的 tag/ref 校验，防止非法 Docker tag、semver alias 覆盖、非默认分支乱发正式 tag；构建阶段使用 Docker Hub environment gate、Buildx，多架构 build 后按 digest 上传并合并 manifest。

来源: `.github/workflows/docker-publish.yml#L1-L39`  
来源: `.github/workflows/docker-publish.yml#L40-L75`  
来源: `.github/workflows/docker-publish.yml#L126-L176`  
来源: `.github/workflows/docker-publish.yml#L194-L200`

### Q8：不启动 MySQL、Redis、WuKongIM 能直接跑起来吗？

不能按完整功能运行。Quickstart 的本地构建前提是有可达 WuKongIM、MySQL 8、Redis 7，可选对象存储。启动期 MySQL ping、session Redis Lua probe、migration/兼容处理都会执行；这些关键依赖失败会 panic。

来源: `QUICKSTART.md#L52-L58`  
来源: `pkg/db/mysql.go#L18-L35`  
来源: `main.go#L214-L227`  
来源: `main.go#L454-L486`

## 5. 边界易错点

1. **不要把 `go build ./...` 当成产出二进制。** 要可运行文件用 `go build -o octo-server .`。
2. **不要把本仓 Dockerfile 当成全栈部署。** 它只构建 server 镜像，全栈在 octo-deployment。
3. **不要推荐旧 `make push/deploy` 作为正式发布。** 文档明确说它们是历史遗留。
4. **不要说 Dockerfile.ghcr 会编译 Go。** 它拷贝预构建二进制。
5. **不要忽略外部依赖。** server 本体运行需要 MySQL/Redis/WuKongIM 等可达服务。

## 6. 不确定部分

- 本卡没有展开 octo-deployment 仓库内部 compose/Helm 配置；如考官追问全栈端口、环境变量和 init 容器细节，需要补 octo-deployment 源码证据。
- 本卡没有跑实际 Docker build；只基于目标仓库构建文件与文档做源码级说明。
