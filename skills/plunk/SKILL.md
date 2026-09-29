---
name: plunk
description: |
  Plunk 开发总入口（路由 + 术语 + 入门引导）。当问题落在这 11 个域时使用：营销活动与统计口径（打开率、点击率、sentCount、投诉率）；Webhook 出站与接收端设计（payload、10 秒超时、SSRF、要不要自建重试队列）；事务邮件 /v1/send 与附件；事件追踪 /v1/track 与打开点击追踪模式；数据保留（正文 90 天、请求日志 30 天、要不要自建归档）；计费 credits 与四类上限（402、超限后入站静默丢弃）；退信与投诉分级、列表卫生、SES 模拟器演练；分群与过滤器 DSL（动态/静态、操作符、refresh/compute）；订阅状态、退订与偏好页、snooze；端到端配方（waitlist、double opt-in、退订双向同步）；MCP 与 AI agent 集成。同时负责把意图路由到 7 个已晋级 Skill（域名 DNS、API 密钥与幂等、联系人建模、模板 Liquid、工作流编排、入站邮件、自托管运维）。triggers: webhook, 打开率, 点击率, 活动统计, 90 天, 数据保留, MCP, Cursor, Claude, 事务邮件, /v1/send, /v1/track, 退信, bounce, 投诉, complaint, 分群, segment, 过滤条件, 退订, snooze, waitlist, double opt-in, credits, 402, 配额, billing limit
metadata:
  cangjie.generated-by: cangjie-tools v2.5.0
  cangjie.variant: router
  cangjie.bundle-id: bundle.plunk-docs
  cangjie.capability-count: 18
  cangjie.entrypoint-count: 8
---
# Plunk Developer Docs — 来源路由入口（compact pack）

## 触发与不触发

**适用**：与本书能力域相关的咨询与任务（见下方路由表的意图列）。
**不适用**：
- Plunk 之外的邮件服务商（SES 直连、SendGrid、Resend 等）
- 邮件文案撰写与营销内容策略
- 非 Plunk 托管的自研 SMTP 服务器调优

## 核心原则（常驻速览，概览类问题读到这里即可回答）

1. Contact 是唯一一等公民：data.* 同时驱动模板个性化、分群过滤与工作流条件
2. 三条发送通道（/v1/send、Campaign、Workflow）共用模板与订阅规则
3. 订阅状态是硬约束：营销/headless 模板尊重 opt-out，transactional 绕开
4. 过滤器 DSL 三处同构：segment condition、campaign audienceCondition、workflow CONDITION
5. 可投递性是一等公民：域名必须验证，硬退信与投诉会自动退订
6. 幂等与错误信封是 API 契约的一部分

## 能力路由（先读本表，按意图加载 1 张能力卡）

| 用户意图 | 先读 | 补读/备注 |
|---|---|---|
| 接入 Plunk API 并处理认证；设计幂等重试；解析错误信封并排障；遍历列表与批量端点 | references/capabilities/plunk-api-essentials.md | 已晋级为独立 Skill `plunk-api-essentials`（已安装时优先直接使用；本卡仅作原文与背景补充） |
| 设计联系人自定义字段 schema；修复字段类型污染；批量导入联系人名单；安全下线一个自定义字段 | references/capabilities/plunk-contacts-and-fields.md | 已晋级为独立 Skill `plunk-contacts-and-fields`（已安装时优先直接使用；本卡仅作原文与背景补充） |
| 准备发信域名；排查邮件进垃圾箱；排查 DNS 记录长期未验证 | references/capabilities/plunk-domain-deliverability.md | 已晋级为独立 Skill `plunk-domain-deliverability`（已安装时优先直接使用；本卡仅作原文与背景补充） |
| 为域名启用收件；做收件自动回复；按收件地址分派下游；排查收不到信或被静默丢弃 | references/capabilities/plunk-inbound-email.md | 已晋级为独立 Skill `plunk-inbound-email`（已安装时优先直接使用；本卡仅作原文与背景补充） |
| 首次自托管部署 Plunk；配置 AWS SES 与 SNS 事件回传；排查自托管发不出信；规划限流与环境变量；自建计费与反钓鱼开关 | references/capabilities/plunk-self-hosting.md | 已晋级为独立 Skill `plunk-self-hosting`（已安装时优先直接使用；本卡仅作原文与背景补充） |
| 选择模板类型；排查变量没渲染出来；纠正条件分支走反；做多语言模板 | references/capabilities/plunk-templates-liquid.md | 已晋级为独立 Skill `plunk-templates-liquid`（已安装时优先直接使用；本卡仅作原文与背景补充） |
| 编排多段邮件序列；排查 workflow 不触发；在活跃执行期安全改结构；按条件分支路由 | references/capabilities/plunk-workflows.md | 已晋级为独立 Skill `plunk-workflows`（已安装时优先直接使用；本卡仅作原文与背景补充） |
| 判断一封邮件能否到达已退订者；布置退订与偏好页出口；用 snooze 降低投诉率；把退订同步回自己的库 | references/capabilities/plunk-subscription-consent.md | references/capabilities/plunk-sending-health.md、references/capabilities/plunk-templates-liquid.md |
| 估算月度 credit 与账单；设计四类分类上限；处理超限后的 402 与入站丢失；排查订阅激活失败 | references/capabilities/plunk-billing-limits.md | references/capabilities/plunk-inbound-email.md |
| 创建并发送一次性群发；解读活动统计；判断误发后如何止损 | references/capabilities/plunk-campaigns-analytics.md | references/capabilities/plunk-templates-liquid.md、references/capabilities/plunk-segments-filters.md |
| 把 Plunk MCP 接入 Claude 或 Cursor；给 agent 设只读或最小权限；排查 agent 发不出邮件；让 agent 起草并发送活动 | references/capabilities/plunk-mcp-agents.md | references/capabilities/plunk-api-essentials.md |
| 搭 waitlist 自动确认邮件；实现双重 opt-in；把订阅状态双向同步回自家数据库；决定 workflow 拆分粒度 | references/capabilities/plunk-recipes.md | references/capabilities/plunk-workflows.md |
| 把受众描述翻译成过滤条件树；选择动态还是静态分群；排查分群人数不对；用分群进出事件驱动自动化 | references/capabilities/plunk-segments-filters.md | references/capabilities/plunk-contacts-and-fields.md、references/capabilities/plunk-campaigns-analytics.md |
| 分级处理退信与投诉；演练退信投诉链路；做列表卫生 | references/capabilities/plunk-sending-health.md | references/capabilities/plunk-subscription-consent.md |
| 上报业务事件；选择打开点击追踪模式；修正打点副作用 | references/capabilities/plunk-tracking-events.md | references/capabilities/plunk-contacts-and-fields.md |
| 发送事务邮件；给邮件加附件；用 emailId 关联 webhook | references/capabilities/plunk-transactional-send.md | references/capabilities/plunk-api-essentials.md |
| 把 Plunk 事件转发到自有服务；用 emailId 对齐发送请求；设计可靠的 webhook 接收端 | references/capabilities/plunk-webhooks.md | references/capabilities/plunk-workflows.md |
| 判断某类数据还能不能查到；设计发送内容归档方案；配置自托管清理作业与 TZ；评估升级后的首次批量清理 | references/capabilities/plunk-data-retention.md | references/capabilities/plunk-self-hosting.md |

**非能力类查询**：
- 书名/作者/章节/整书概览 → references/overview.md
- 术语解释 → references/glossary.md
- 决策规则速查（不需要原文依据时） → references/cheatsheet.md
- 完整意图与关键词索引（本表未覆盖的意图先查这里） → references/capability-index.md

## 加载规则

- 每次任务先读本文件，再按路由表加载 **1** 张能力卡；任务明确跨域时最多加载 2 张。
- 概览/书名类问题不加载能力卡，用「核心原则」与 overview.md 回答。
- 路由表与 capability-index.md 都无法命中的意图，明确告知超出本书范围，不要硬套。

## 边界与判停

- 发信域名未验证时先停下去做 /guides/verifying-domains，不要继续构造发送请求
- 文档自身口径冲突的项（限流、webhook 重试、保留键清单）并列呈现，不替文档取舍
- Workflow 的 API 创建缺少完整 schema，改走 dashboard/MCP 或探测 /workflows/fields
