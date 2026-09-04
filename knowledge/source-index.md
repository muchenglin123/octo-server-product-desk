# Source Index

本文件用于索引 octo-server 知识库中的 9 个知识块。

Agent 回答问题时，应优先查看对应知识文件，并在回答中引用源码路径和行号。

## 知识文件索引

| 编号 | 知识块 | 文件 |
|---|---|---|
| 01 | 认证与身份 | knowledge/01-auth-identity.md |
| 02 | 鉴权模型 | knowledge/02-authorization-model.md |
| 03 | 配置 | knowledge/03-configs.md |
| 04 | 业务模块清单 | knowledge/04-modules.md |
| 05 | API 与错误约定 | knowledge/05-api-errors.md |
| 06 | IM 控制面 | knowledge/06-im-control-plane.md |
| 07 | Bot 与 Agent | knowledge/07-bot-agent.md |
| 08 | 存储与外部依赖 | knowledge/08-storage-dependencies.md |
| 09 | 构建与发布 | knowledge/09-build-release.md |

## 使用规则

1. 产品问题先判断属于哪个知识块。
2. 到对应知识文件中查找结论。
3. 回答时必须带源码引用。
4. 引用格式必须是：

来源: <相对路径>#L<起>-L<止>

5. 如果对应知识文件没有内容，必须回答：

我不确定。当前知识库没有覆盖这个问题，需要补充对应模块的源码分析。

6. 不允许编造路径。
7. 不允许编造行号。
8. 不允许把 README 里的描述当作源码证据。
9. 源码证据必须来自目标仓库 octo-server。

## 待补充

后续需要把每个知识文件中的“待补充”部分，替换为真实源码分析和可核验引用。
