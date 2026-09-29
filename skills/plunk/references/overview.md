# Plunk 官方文档 · 书级概览（随包分发）

> 完整版见 `BOOK_OVERVIEW.md`（流水线全局上下文）。本文件是随 Skill 分发的精简版：骨架、核心命题、能力清单、文档边界。

## 来源

- https://docs.useplunk.com/ 全站 48 页，2026-09-29 抓取，347,651 字符
- 作者：Plunk 团队（开源仓库 useplunk/plunk，MIT）

## 五层结构

1. **概念层 concepts/**（7 篇）：Contacts、Segments、Templates、Campaigns、Workflows、Transactional emails、Billing
2. **指南层 guides/**（15 篇）：API keys、Verifying domains、Tracking、Webhooks、Receiving emails、Custom fields、Segment filters、Importing contacts、Unsubscribe & preferences、Localization、List hygiene、Template language、Idempotency、Data retention、MCP server
3. **API 参考层 api-reference/**（17 篇）：Public API 3 个（`/v1/send`、`/v1/track`、`/v1/verify`）+ 资源 API + 错误码总表
4. **配方层 recipes/**（4 篇）：waitlist、double opt-in、sync unsubscribes
5. **自托管层 self-hosting/**（4 篇）：Docker Compose、AWS SES、环境变量全集、引言

## 核心命题

1. **Contact 是唯一一等公民。** 一切都挂在 contact 上：`data.*` 自定义字段同时驱动个性化（模板变量）和分群（segment filter）；`/v1/track` 打事件会自动 upsert contact；inbound 邮件会自动建 contact。
2. **三条发送通道，共用一套模板与订阅规则。** `/v1/send`（一对一事务）、Campaign（一对多群发，一次性）、Workflow（自动化编排，多步骤）。三者都消费 Template，都受"模板类型 × 订阅状态"矩阵约束。
3. **订阅状态是硬约束，不是软建议。** MARKETING / HEADLESS 尊重 opt-out；TRANSACTIONAL 绕开。即使走 `/v1/send` 用 marketing 模板，也到不了已退订的人。
4. **事件是自动化的燃料。** `/v1/track` 的自定义事件 + 系统事件（`email.*`、`contact.subscribed/unsubscribed`、`segment.<slug>.entry/exit`、`email.received`）既是 workflow 触发器，也是 segment 过滤维度。
5. **过滤器 DSL 三处同构。** 动态 segment 的 `condition`、campaign 的 `audienceCondition`（`FILTERED`）、workflow 的 `CONDITION` 步骤，共用 `{logic, groups[].filters[]}` 结构与同一套操作符。学会一次，到处能用。
6. **可投递性是一等公民，不是附加题。** 发信域名必须验证（DKIM 3 CNAME + SPF 1 TXT + bounce MX + 建议 DMARC）；硬退订与投诉会自动退订联系人；系统自动生成 plain text 替代件以规避 HTML-only 垃圾信号；snooze 作为"暂停而非永久退出"压抑投诉率。
7. **计费按封数、按类别可设上限。** 出站（事务/活动/工作流各 1 credit）、入站 1 credit、带附件 2 credits；四个类别可独立设 cap；超限对外 402，对入站静默丢弃。
8. **幂等与错误处理是契约的一部分。** `Idempotency-Key` 在 `/v1/send` 与 `/v1/track` 上提供 at-most-once；错误响应统一为 `{success,error{code,message,statusCode,requestId,errors[],suggestion},timestamp}`。

## 能力清单（18）

| slug | 标题 | 去向 | 重要性 | 一句话 |
| --- | --- | --- | --- | --- |
| `plunk-api-essentials` | API 调用基石：密钥、错误、幂等与限流 | 晋级 Skill | critical | 按 error.code 分支而不是 HTTP 状态码，2xx/5xx 占用幂等键、4xx 才释放 |
| `plunk-contacts-and-fields` | 联系人建模与自定义字段 | 晋级 Skill | critical | 清空字段要传 null 而不是空字符串，空串会被静默忽略 |
| `plunk-domain-deliverability` | 发信域名与 DNS 认证 | 晋级 Skill | critical | 一个域名只能有一条 SPF 记录，要用合并而不是新增 |
| `plunk-inbound-email` | 入站邮件处理 | 晋级 Skill | critical | 入站判定只是记录不拦截，过滤与防回环必须自己在 workflow 里做 |
| `plunk-self-hosting` | 自托管部署与运维 | 晋级 Skill | critical | 不跑 worker 的部署，保留任务永不执行、数据无界增长 |
| `plunk-templates-liquid` | 模板与 Liquid 渲染 | 晋级 Skill | critical | HEADLESS 尊重退订但不给页脚，必须自己在正文放 {{unsubscribeUrl}} |
| `plunk-workflows` | 工作流编排 | 晋级 Skill | critical | 建完默认 enabled:false 且 triggerType 恒为 EVENT，不改就不会有执行 |
| `plunk-subscription-consent` | 订阅状态、退订与合规 | 总入口内能力卡 | critical | 走 /v1/send 也豁免不了订阅检查，判据是模板类型不是通道 |
| `plunk-billing-limits` | 计费与限额 | 总入口内能力卡 | high | 出站超限返回 402，入站超限静默丢弃且永不重放 |
| `plunk-campaigns-analytics` | 营销活动与统计口径 | 总入口内能力卡 | high | 三个损失列互斥，硬退与投诉绝不进 unsubscribedCount |
| `plunk-mcp-agents` | MCP 与 AI agent 集成 | 总入口内能力卡 | high | 自托管接 MCP 必须用 PLUNK_MCP_API_KEY，否则 agent 会静默操作错的项目 |
| `plunk-recipes` | 端到端配方 | 总入口内能力卡 | high | 确认邮件必须用 TRANSACTIONAL 模板，marketing 模板对未订阅联系人会静默跳过 |
| `plunk-segments-filters` | 分群与过滤器 DSL | 总入口内能力卡 | high | 组内恒为 AND，要表达"或"必须新开一个 group 挂 conditions |
| `plunk-sending-health` | 退信、投诉与列表卫生 | 总入口内能力卡 | high | 只有 Permanent 硬退会自动退订，软退只做可见性 |
| `plunk-tracking-events` | 事件追踪 | 总入口内能力卡 | high | /v1/track 会自动建联系人，新联系人默认订阅 |
| `plunk-transactional-send` | 事务邮件发送 | 总入口内能力卡 | high | 内容二选一：给 template，或者 subject 与 body 齐备 |
| `plunk-webhooks` | Webhook 出站集成 | 总入口内能力卡 | high | 自定义 body 会整体替换默认 payload，连 emailId 一起没了 |
| `plunk-data-retention` | 数据保留 | 总入口内能力卡 | medium | 90 天后正文被清空且不可逆，要存档只能在发送侧自己留 |

## 文档自身的边界（不得写成确定性做法）

- **needs_review**：保留在交付里，但以"警示 / 边界 / 待验证"形态出现，不承担方法主体的可执行性。
- **rejected**：与其他单元重复，不进入交付主体，`rejected/` 仅留审计痕迹。

> 完整清单：`needs-review.md`（12 组口径冲突 / 瑕疵 / 能力缺口）、`references.md`（只读查询表与不交付项）。
