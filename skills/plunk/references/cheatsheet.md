# 决策规则速查 — Plunk Developer Docs

| 能力 | 一句话规则 |
|---|---|
| API 调用基石：密钥、错误、幂等与限流 | 按 error.code 分支而不是 HTTP 状态码，2xx/5xx 占用幂等键、4xx 才释放 |
| 联系人建模与自定义字段 | 清空字段要传 null 而不是空字符串，空串会被静默忽略 |
| 发信域名与 DNS 认证 | 一个域名只能有一条 SPF 记录，要用合并而不是新增 |
| 入站邮件处理 | 入站判定只是记录不拦截，过滤与防回环必须自己在 workflow 里做 |
| 自托管部署与运维 | 不跑 worker 的部署，保留任务永不执行、数据无界增长 |
| 模板与 Liquid 渲染 | HEADLESS 尊重退订但不给页脚，必须自己在正文放 {{unsubscribeUrl}} |
| 工作流编排 | 建完默认 enabled:false 且 triggerType 恒为 EVENT，不改就不会有执行 |
| 订阅状态、退订与合规 | 走 /v1/send 也豁免不了订阅检查，判据是模板类型不是通道 |
| 计费与限额 | 出站超限返回 402，入站超限静默丢弃且永不重放 |
| 营销活动与统计口径 | 三个损失列互斥，硬退与投诉绝不进 unsubscribedCount |
| MCP 与 AI agent 集成 | 自托管接 MCP 必须用 PLUNK_MCP_API_KEY，否则 agent 会静默操作错的项目 |
| 端到端配方 | 确认邮件必须用 TRANSACTIONAL 模板，marketing 模板对未订阅联系人会静默跳过 |
| 分群与过滤器 DSL | 组内恒为 AND，要表达"或"必须新开一个 group 挂 conditions |
| 退信、投诉与列表卫生 | 只有 Permanent 硬退会自动退订，软退只做可见性 |
| 事件追踪 | /v1/track 会自动建联系人，新联系人默认订阅 |
| 事务邮件发送 | 内容二选一：给 template，或者 subject 与 body 齐备 |
| Webhook 出站集成 | 自定义 body 会整体替换默认 payload，连 emailId 一起没了 |
| 数据保留 | 90 天后正文被清空且不可逆，要存档只能在发送侧自己留 |

> 速查只给结论；需要原文依据、案例或反例时读对应能力卡。
