# octo-server 产品需求池

这是 AINOL Agent 实操考核使用的 octo-server 产品需求池仓库。

## 目标项目

目标项目：octo-server  
目标仓库：https://github.com/Mininglamp-OSS/octo-server

注意：目标仓库只读，禁止写入。

## 本仓库用途

本仓库用于归档和管理 octo-server 相关的：

- Bug 反馈
- Feature 需求
- 产品问题
- PRD 文档
- Review 意见
- Agent 定时巡检记录

## Agent 设计

本次考试设计一个核心 Agent：

**octo-server 产品管家**

它负责：

1. 回答 octo-server 产品和功能问题
2. 基于源码证据给出结论
3. 识别群里的 Bug / Feature / Question
4. 将需要归档的内容创建为 GitHub issue
5. 给 issue 打类型、优先级、状态和产品域标签
6. 为 Feature 补充 PRD
7. 定时扫描 GitHub issue 变化
8. 有实质变化时回 Octo 考试群汇报

## 标签体系

### 类型标签

- type/bug：Bug 反馈
- type/feature：新功能需求
- type/question：产品问题
- type/prd：PRD 文档相关

### 优先级标签

- priority/P0：严重阻断
- priority/P1：高优先级
- priority/P2：普通优先级
- priority/P3：低优先级或优化项

### 状态标签

- status/new：新收到
- status/triaged：已分诊
- status/need-info：需要补充信息
- status/prd-drafting：正在补充 PRD
- status/in-review：等待评审
- status/changes-requested：评审打回，需要修改
- status/accepted：已通过
- status/wontfix：确认不做
- status/closed：已关闭

### 产品域标签

- area/auth：认证与身份
- area/rbac：鉴权模型
- area/config：配置
- area/api：API 与错误约定
- area/im：IM 控制面
- area/bot-agent：Bot 与 Agent
- area/storage：存储与外部依赖
- area/build-release：构建与发布
- area/module：业务模块

## 群内汇报规则

Agent 主动回 Octo 考试群时必须遵守：

1. 有实质变化才发消息
2. 没有变化保持沉默
3. 不发送“正在检查”
4. 不发送“本次无更新”
5. 不发送“一切正常”
6. 每条主动汇报都要 @ 主考
7. 如有对应提交人或负责人，也要一起 @

## PRD 规则

PRD 只写 What，不写 How。

允许写：

- 用户遇到什么问题
- 用户希望完成什么
- 用户能感知到什么结果
- 功能范围
- 非目标范围
- 用户视角的验收标准

禁止写：

- Redis
- 数据库表
- 接口字段
- 代码实现
- 缓存方案
- HTTP 200
- 内部字段名
- 代码块

## 源码引用规则

Agent 回答 octo-server 产品问题时，关键结论必须带源码引用。

引用格式：

```text
来源: <相对路径>#L<起>-L<止>
