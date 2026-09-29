---
name: plunk-inbound-email
description: |
  当你要让 Plunk 接收发往已验证域名的邮件并驱动自动化（自动回复、工单、按 support/sales/billing 分流），
  或收不到信、邮件进了 Activity 却不跑 workflow、来信被静默丢弃、受众被陌生发件人污染时使用。
  用户提供收信域名、DNS 现状与期望的下游动作，产出四步启用流程（含子域冲突解法）、带护栏的 workflow 设计与四级/三查排障路径。
  不负责发信域名的 DNS 认证细节、workflow 图结构本身与 webhook 出站契约；
  附件、原始 MIME 与线程一律不保留，>40MB 直接拒收。
  triggers: inbound, 收件, 收信, "email.received", "inbound MX", spamVerdict, virusVerdict, auto-reply, catch-all, "reply loop", "40 MB", attachments, inbound cap
metadata:
  cangjie.generated-by: cangjie-tools v2.5.0
  cangjie.capability-id: cap.plunk-docs.inbound-email
  cangjie.capability-revision: 1
  cangjie.bundle-id: bundle.plunk-docs
  cangjie.source-title: Plunk Developer Docs
  cangjie.tags: inbound, email-receiving, dns, troubleshooting, security
---
# 入站邮件处理

## R 原文依据

- [/guides/receiving-emails](https://docs.useplunk.com/guides/receiving-emails)："Inbound is only enabled on domains that are fully verified for sending (DKIM + SPF + the bounce-feedback MX)."
- [/guides/receiving-emails](https://docs.useplunk.com/guides/receiving-emails)："A domain can only have one primary inbound MX target. If you already use Google Workspace, Microsoft 365... your existing email will break if you switch the MX to point at Plunk. The usual fix is to receive Plunk inbound on a subdomain... The subdomain still needs to be verified for sending in Plunk before its MX will be accepted."
- [/guides/verifying-domains](https://docs.useplunk.com/guides/verifying-domains)："This MX record handles delivery feedback (bounces, complaints) for emails Plunk sends out — it's separate from the inbound MX record used to **receive** emails sent to your domain. They serve different purposes; you can have either one or both."
- [/guides/receiving-emails](https://docs.useplunk.com/guides/receiving-emails)："1. Parses the message and stores it as an inbound `Email` record... 2. Creates the sender as a contact in your project (or updates them if they already exist), **subscribed by default**. 3. Tracks an `email.received` event on the sender contact."
- [/guides/receiving-emails](https://docs.useplunk.com/guides/receiving-emails)："The event is **always** emitted — Plunk does not block emails based on the spam, virus, or authentication verdicts. Filter them yourself in the workflow that fires."
- [/guides/receiving-emails](https://docs.useplunk.com/guides/receiving-emails)："Plunk does not store: the original raw message, attachments, threading headers (`In-Reply-To`, `References`), or headers beyond what's exposed on the event."；"**40 MB size cap**: messages larger than 40 MB are rejected before processing."
- [/guides/receiving-emails](https://docs.useplunk.com/guides/receiving-emails)："Once the cap is reached, **inbound emails are silently dropped for that project** until the cap resets — they aren't queued or replayed. Other projects sharing the same domain are unaffected by another project's cap."
- [/guides/receiving-emails](https://docs.useplunk.com/guides/receiving-emails)："**Reply-loop risk.** If your auto-reply sends back to a domain you also receive on (or to a list address), you can create an infinite loop. Add a `CONDITION` that drops messages where `event.from` matches your own domain."

## I 解释

Plunk 的入站不是邮箱服务，而是"把发往已验证域名的邮件翻译成一个事件"。它只做三件事：把邮件解析消毒后存成一条 inbound Email 记录、把发件人 upsert 成**已订阅**联系人、在该联系人上打一个 `email.received` 事件。之后一切都交给 workflow——收件后的路由、过滤、回复、转发，全靠你自己编排。

前置条件被设计得很硬：域名必须先**完整**通过发信验证（DKIM + SPF + bounce 反馈 MX），未验证的域名上 inbound MX 完全不处理任何邮件。而 Plunk 里有两条用途完全不同的 MX——bounce MX 收的是"你发出去的邮件的投递反馈"（退信、投诉），inbound MX 收的是"别人发给你的邮件"，两者互相独立、可只配其一。混淆它们会导致"以为配了就能收信"。

因为一个域只能有一个主收信 MX，冲突是必然场景：顶域已经在用 Workspace / M365 收信时，把 MX 指向 Plunk 会让现有邮箱彻底挂掉。解法是把 Plunk 收信放到子域，且该子域同样要完成发信验证。

过滤责任的划分是这套设计里最容易被误解的一点：spam / virus / SPF / DKIM / DMARC 判定只是**记录在事件上的建议**，Plunk 不基于它们阻断任何邮件，事件永远会发出。所以"Plunk 会帮我拦垃圾邮件"是错的——你必须在 workflow 的第一个 CONDITION 丢弃 `spamVerdict` / `virusVerdict` 为 FAIL 的邮件。同理，发件人被自动创建为 subscribed 联系人，若不想让外部来件人进你的营销受众，要在流程末尾用 UPDATE_CONTACT 设 `subscribed: false`。

两个行为不对称要特别记住：入站是 catch-all（任意地址都进同一处理器，靠 `event.to` 在 workflow 内分流），以及同域在多个项目验证时是**扇出**而非争抢（每个项目各得一份 Email 记录、各 upsert 一次、各发一个事件）。另外计费超限的表现也不同：出站超限会拿到 402 `BILLING_LIMIT_EXCEEDED`，入站触顶则是**静默丢弃、不排队不重放**——没有报错，所以只能靠定期查 Billing → Limits 发现。

## A1 示例

**示例 1 · 最小自动回复**（来自 `/guides/receiving-emails`）

- 输入/问题：`support@yourdomain.com` 收到信后要自动回一条确认。
- 怎么执行：① Trigger `email.received`；② CONDITION：仅当 `event.to == support@yourdomain.com` 且 `event.spamVerdict == "PASS"` 且 `event.virusVerdict == "PASS"` 才继续；③ SEND_EMAIL：To `{{event.from}}`、Subject `Re: {{event.subject}}`、固定正文。文档注明"Use a `TRANSACTIONAL` template for the auto-reply so it bypasses subscription checks"。
- 产出/预期输出：文档未说明（原书未给出执行结果）。

**示例 2 · 按收件地址路由分发**（来自 `/guides/receiving-emails`）

- 输入/问题：support / sales / billing 三个地址要落到不同下游系统。
- 怎么执行：Trigger `email.received`，CONDITION 按 `event.to` 分支——`support@…` → ticketing webhook → auto-reply；`sales@…` → CRM webhook → notify Slack；`billing@…` → billing system webhook。
- 产出/预期输出：文档未说明（原书未给出执行结果）。

**示例 3 · 转发到自有 API 补偿能力缺口**（来自 `/guides/receiving-emails`）

- 输入/问题：需要分类、建工单，或需要 Plunk 未保留的内容。
- 怎么执行：Trigger `email.received` → WEBHOOK 步骤，`URL: https://api.example.com/inbound`、Method `POST`，body 为 `{from, subject, body, messageId, timestamp, verdicts: {spam, virus, spf, dkim, dmarc}}`，各值用 `{{event.*}}` 插值。
- 产出/预期输出：文档未说明（原书未给出执行结果）。注意：附件在 Plunk 侧解析时已被丢弃，转发也转不出去，需要附件得在 SES 侧另接。

**示例 4 · 前置过滤 + 防回环**（来自 `/guides/receiving-emails`）

- 输入/问题：既要拦垃圾/病毒，又要防止自动回复自己回给自己形成死循环。
- 怎么执行：workflow 靠前的 CONDITION 丢弃 `spamVerdict` 或 `virusVerdict` 为 `FAIL` 的邮件；再加一个 CONDITION 丢弃 `event.from` 属于自己域名的邮件。
- 产出/预期输出：文档未说明（原书未给出执行结果）。可判定的是两条护栏都生效后不再出现互发循环。

## A2 何时会用

**用户场景**

1. 想让 Plunk 接收发往自己域名的邮件，并触发自动回复或工单。
2. 配了 MX 但收不到信，或邮件进了 Activity 却没有 workflow 跑。
3. 顶域已经在用 Google Workspace / M365 收信，担心切 MX 会弄挂现有邮箱。
4. 要把 support@ / sales@ / billing@ 分派到不同下游系统。
5. 开收信后发现营销受众里混进了大量陌生发件人、收到了明显是垃圾/带毒的邮件，或来信突然"无人来信"、怀疑被限额丢掉了。

**语言信号**

`收件` / `收信` / `inbound` / `receiving emails` / `email.received` / `MX record` / `inbound MX` / `自动回复` / `auto-reply` / `catch-all` / `spamVerdict` / `virusVerdict` / `SPF` / `DKIM` / `DMARC` / `verdict` / `工单` / `ticketing` / `回环` / `reply loop` / `40 MB` / `附件` / `attachments`

**与相邻能力的区分**

- 本卡负责入站链路：启用条件（域名完整发信验证 + inbound MX）、`email.received` 事件、安全判定、路由编排与收不到信的排查。
- `plunk-domain-deliverability` 负责发信域名的 DNS 认证四件套（含 bounce MX）——本卡依赖它做前置，但不重复讲 SPF/DKIM/DMARC 怎么配。
- `plunk-workflows` 负责 workflow 图本身（trigger、step、branch、执行状态）——本卡只把 `email.received` 当作已知触发器来用。
- `plunk-webhooks` 负责把入站邮件转发到自有 API 时的出站契约（超时、重试、SSRF）。
- `plunk-subscription-consent` 负责订阅状态与合规——本卡只在"发件人被自动订阅"这一点上与它交接。
- `plunk-billing-limits` 负责限额分类与 credits 口径——本卡只引用"入站触顶静默丢弃"这一条后果。

## E 执行步骤

**输入契约（必需信息）**

1. 用于收信的域名（或子域）及其 DNS 管理权限。
2. 该域当前是否已在用其他服务商收信（决定要不要走子域）。
3. 收信后要做什么：自动回复、转发到哪个 API、按哪些地址分流。
4. 缺失时先问：① 该域是否已完成发信验证（DKIM + SPF + bounce MX）；② 是否有不想进入营销受众的发件来源；③ 是否需要附件/线程（若需要，本能力直接不适用）。

**步骤**

1. **确认前置：域名完整验证为可发信**。入站只在"完全验证为可发信"的域名上启用（DKIM + SPF + bounce 反馈 MX）。完成标准：dashboard 里该域显示为已验证；未验证时 inbound MX 完全不处理邮件。若还没做，先转 `plunk-domain-deliverability` 的流程。
2. **区分并加 inbound MX**：打开项目 Domains 页，展开已验证域，在 **Inbound Email** 区块复制 Plunk 给出的 MX 值加到 DNS。先分清两条 MX：

   | | bounce MX（域名验证四件套之一） | inbound MX（本卡） |
   | --- | --- | --- |
   | 用途 | 接收 Plunk **发出**邮件的投递反馈（退信、投诉） | **接收**发往该域名的邮件 |
   | 是否必做 | 是 | 可选，启用收信时才加 |
   | 取值来源 | 域名验证流程给出 | Domains 页 Inbound Email 区块给出 |
   | 生效前提 | 域名验证通过 | 域名**完整**通过发信验证 |

   两者可以只配其一，也可以都配。完成标准：DNS 里存在 inbound MX 记录；不要以为配了 bounce MX 就能收信。⚠️ 文档口径：inbound MX 的具体取值只能在 dashboard 的 Domains 页读取，Billing → Limits 的 cap 与项目状态也只在 dashboard 里看——这些是 dashboard-only 操作，文档未给出对应 API，不要编造端点。
3. **若顶域已有邮箱在用（Workspace / M365 等）**：不要改顶域 MX——一个域只能有一个主收信 MX，改了现有邮箱会挂。改用子域（如 `mail.yourdomain.com` / `support.yourdomain.com`）接 Plunk 入站，且该子域同样要在 Plunk 完成发信验证，其 MX 才会被接受。完成标准：子域已验证并带上 inbound MX，顶域 MX 未动。
4. **等传播并验证**：DNS 变更从几分钟到 48 小时不等，用 `dig MX yourdomain.com` 确认返回值里有 Plunk 给你的那个值。完成标准：dig 输出命中。
5. **发外部测试信并三点验收**：从任意外部账号发到该域任意地址，检查 ① Activity 出现一条 type 为 **Inbound** 的 Email 行；② 发件人出现在 Contacts；③ 该联系人上记录了 `email.received` 事件。完成标准：三条全中。
6. **认清 `email.received` 事件的字段**：编排时只能用到事件载荷里暴露的这些字段——

   | 字段 | 说明 |
   | --- | --- |
   | `messageId` | 该邮件的唯一标识 |
   | `from` | 发件人地址（envelope / `From` 头） |
   | `fromHeader` | 完整 `From` 头，含显示名（如 `"Ada <ada@example.com>"`） |
   | `to` | 你在已验证域上的主收件地址 |
   | `recipients` | 全部收件地址数组（覆盖 `To` / `Cc` 与 envelope BCC） |
   | `subject` | 主题 |
   | `timestamp` | ISO 8601 接收时间 |
   | `hasContent` | 是否成功解析出正文 |
   | `body` | 消毒后的 HTML 正文，无 HTML 时为纯文本 |
   | `spamVerdict` / `virusVerdict` | 垃圾 / 病毒判定，`PASS` / `FAIL` / `GRAY` / `PROCESSING_FAILED` |
   | `spfVerdict` / `dkimVerdict` / `dmarcVerdict` | 三项认证结果（文档未给枚举） |
   | `processingTimeMillis` | 处理耗时（毫秒） |

   模板与 workflow 步骤里用 `{{event.<key>}}` 引用。完成标准：需要的字段都在上表内；需要上表外的信息（附件、原始 MIME、其它头）则本能力不适用。

7. **建 workflow 并加护栏**：Trigger 用 `email.received`（名字必须精确）。第一个 CONDITION 丢弃 `spamVerdict` 或 `virusVerdict` 为 `FAIL` 的邮件——Plunk 不做预过滤，这一步必须自建。注意 `GRAY` 与 `PROCESSING_FAILED` 都不是 `PASS`，严格过滤时应视为不通过。完成标准：一条明显垃圾邮件进来时不产生下游动作。
8. **按收件地址分流**：Plunk 是 catch-all，任意地址都进同一处理器，路由只能在 workflow 内用 CONDITION 对 `event.to` 分支（如 support → 工单 webhook + 自动回复；sales → CRM webhook + Slack 通知；billing → 计费系统 webhook）。完成标准：三个地址各自落到预期下游。
9. **若做自动回复**：用 `TRANSACTIONAL` 模板以绕开订阅检查（发件人虽被自动订阅，但日后退订仍应收到回复）；收件人 `{{event.from}}`、标题 `Re: {{event.subject}}`。**同时**加一条 CONDITION 丢弃 `event.from` 属于自己域名的邮件做回环防护。完成标准：外域来信收到回复；自域来信不再触发回复（否则会形成无限循环）。
10. **处置自动订阅**：若不想让来件人进入营销受众，在 workflow 末尾加 `UPDATE_CONTACT` 设 `subscribed: false`。完成标准：新来件人的订阅状态为 false。
11. **若收不到信，四级排查**：① DNS：`dig MX` 确认记录生效 → ② 域名是否完整验证为可发信（未验证则不处理）→ ③ 建一个"只有 `email.received` trigger + webhook 打点（如 webhook.site）"的最小测试 workflow 来二分定位——有事件则是你的 workflow 逻辑问题，无事件则是上游问题 → ④ 发件方 DNS 缓存，换一个邮箱服务商再测。完成标准：定位到四级之一。
12. **若邮件进了 Activity 但 workflow 不跑，三查**：① trigger 事件名是否精确等于 `email.received` → ② workflow 是否 `enabled`（创建时默认是 false）→ ③ CONDITION 是否把事件全过滤掉了（verdict 检查是最常见元凶，发件域认证宽松时会误杀）。完成标准：定位到三处之一。
13. **若 verdict 持续 FAIL，归因到发件方**：这是发件方配置问题，不是你的——

   | 判定 | FAIL 的常见归因 | 处置 |
   | --- | --- | --- |
   | `spfVerdict` | 对方域名没有 SPF 记录，或未列出其发信 IP | 丢弃或照常处理 |
   | `dkimVerdict` | 对方没有 DKIM，或邮件在传输中被改（部分转发服务会破坏 DKIM） | 丢弃或照常处理 |
   | `dmarcVerdict` | 对方 SPF 与 DKIM 双败，或对齐策略过严被转发破坏 | 丢弃或照常处理 |

   处置是二选一：在 workflow 里丢弃，或照常处理——文档明确说"your call"。完成标准：明确写下本项目的策略（默认是丢弃 spam/virus FAIL；敏感收件箱再叠加丢弃未通过认证的）。
14. **若来信悄悄消失，查丢弃路径**：① 查 Billing → Limits 里 inbound 类别的 cap 是否已到——触顶后该项目静默丢弃、不排队不重放，直到下个周期重置；② 查项目状态，被禁用的项目不处理入站。完成标准：确认是 cap 触顶、项目被禁用，还是都不是。注意这与出站不同：出站超限是 402 `BILLING_LIMIT_EXCEEDED`，入站是静默丢弃、没有任何报错。
15. **若同域在多个项目验证**：接受扇出语义——每个验证了该域的项目各拿一份（各自的 Email 记录、contact upsert、`email.received` 事件）。不想要就一次只在一个项目里验证该域。完成标准：确认该域被哪些项目验证，并确认每个项目的 workflow 不会跨项目互发（尤其是自动回复）。

**输出契约**

- 产出：可用的入站链路（域名 + inbound MX + 至少一条带护栏的 `email.received` workflow）+ 一份排障结论。
- 验收：① `dig MX` 命中 Plunk 给的值；② 外部测试信在 Activity 出现 Inbound 行、发件人成为 contact、该 contact 上有 `email.received`；③ workflow `enabled` 为 true 且目标地址分流正确；④ 自域来信不触发自动回复；⑤ 若涉及限额，Billing → Limits 的 inbound cap 未触顶。
- 交付后必须交代的三件事：① 入站不做任何预过滤，过滤责任在 workflow；② 来件人默认是 subscribed，要不要关由业务定；③ 附件/原始 MIME/线程不可用，需要就得改架构。

## B 边界与失效

1. **不要用 Plunk 入站替代邮箱/工单系统**：硬限制是 catch-all only、无附件（解析时即丢弃）、无原始 MIME、无线程（`In-Reply-To` / `References` 不保留）、单封 > 40 MB 在处理前就被拒。需要这些能力只能转发到自己服务，但附件在 Plunk 侧已经不存在，得在 SES 侧另接。
2. **不要指望 Plunk 帮你拦垃圾/病毒**：事件永远会发出，spam / virus / SPF / DKIM / DMARC 判定只是**建议性**的、不强制执行。直接把 `{{event.body}}` 转进工单系统或渲染到内部 UI，会让恶意内容进入内部系统。
3. **不要把 body 当可信内容**：正文入库前虽已消毒（剥离 script、iframe、事件处理器、`javascript:` URI），仍可能含钓鱼链接、社会工程内容与 unicode 形似字符。渲染到任何你控制的 UI 前必须按该上下文再转义一次。
4. **不要在顶域已有邮箱时改 MX**：一个域只能有一个主收信 MX，改了现有邮箱彻底收不到信。走子域，且子域也要先完成发信验证。
5. **不要忽视自动订阅的合规影响**：所有来件人（含垃圾发件人、外部合作方）都会以 subscribed 状态进入你的受众。若中枢系统不是 Plunk，这一步可能不是你想要的——用 `UPDATE_CONTACT` 设 `subscribed: false`。
6. **不要做无护栏的自动回复**：回给自己也在接收的域（或列表地址）会形成无限循环，烧 credits、触发限流与投诉。回环防护不是可选项。
7. **不要把入站当可靠投递通道**：触顶即静默丢弃、不排队不重放，且文档未给出补回机制。对"必须收到"的业务（如账单回执）不要依赖它；且它与出站的不对称（402 vs 静默）意味着入站没有任何主动告警，只能靠定期检查。
8. **高危不可撤销操作**：切换顶域 MX（现有邮箱中断，恢复需等 DNS 再次传播）、入站邮件本身（一旦因 cap 被丢弃即永久丢失，0 recoverability）、在未加回环防护时上线自动回复（可能造成跨项目/跨域互发）。
9. **不要在 verdict 持续 FAIL 时去改自己的 DNS**：这指向发件方配置问题，改自己这边不会有任何改善。
10. **不要用 `PROCESSING_FAILED` 当 PASS**：spam / virus verdict 取值是 `PASS` / `FAIL` / `GRAY` / `PROCESSING_FAILED`，后两者都不是 PASS；严格过滤时应视为不通过。spf / dkim / dmarcVerdict 的枚举文档未给出，只描述为"认证结果"，不要假设它们与 spam/virus 同枚举。
11. **不要用入站做需要审计留痕的业务**：原始邮件不保留、超 40 MB 直接拒、触顶静默丢弃且不排队不重放，三者叠加意味着"收到的证据"可能根本不存在。合规/审计场景必须在 SES 侧另建通路。
12. **不要让自动回复使用 MARKETING 模板**：自动回复要用 `TRANSACTIONAL` 模板绕开订阅检查——发件人日后退订时，你仍然希望回复能发出去。
