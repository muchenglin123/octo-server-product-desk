# Label 使用说明

本文件说明 octo-server 产品需求池中的标签体系。

## 类型标签

- type/bug：Bug 反馈
- type/feature：新功能需求
- type/question：产品问题
- type/prd：PRD 文档相关

## 优先级标签

- priority/P0：严重阻断，必须优先处理
- priority/P1：高优先级
- priority/P2：普通优先级
- priority/P3：低优先级或优化项

## 状态标签

- status/new：新收到
- status/triaged：已分诊
- status/need-info：需要补充信息
- status/prd-drafting：正在补充 PRD
- status/in-review：等待评审
- status/changes-requested：评审打回，需要修改
- status/accepted：已通过
- status/wontfix：确认不做
- status/closed：已关闭

## 产品域标签

- area/auth：认证与身份
- area/rbac：鉴权模型
- area/config：配置
- area/api：API 与错误约定
- area/im：IM 控制面
- area/bot-agent：Bot 与 Agent
- area/storage：存储与外部依赖
- area/build-release：构建与发布
- area/module：业务模块

## 使用规则

1. 每个 issue 至少要有一个类型标签。
2. Bug 使用 type/bug。
3. 新需求使用 type/feature。
4. 产品问题使用 type/question。
5. 进入 PRD 阶段的需求可增加 type/prd。
6. 每个 issue 建议有一个优先级标签。
7. 每个 issue 建议有一个状态标签。
8. 如果能判断所属模块，应增加对应 area 标签。
9. status/wontfix 表示确认不做，不等于已修复。
10. status/closed 表示 issue 已关闭，不等于问题已解决。
