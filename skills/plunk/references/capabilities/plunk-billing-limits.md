# 计费与限额

## R 原文依据

- [/concepts/billing](https://docs.useplunk.com/concepts/billing)："Transactional sends (`/v1/send`): 1 credit per recipient. Campaigns: 1 credit per recipient. Workflow `SEND_EMAIL` steps: 1 credit per send. Inbound emails: 1 credit per received email. **Emails with attachments**: **2 credits** per email."
- [/concepts/billing](https://docs.useplunk.com/concepts/billing)："Emails with one or more attachments cost double. There's no extra charge based on attachment size — one credit per attached email regardless of whether it's 100 KB or 10 MB."
- [/concepts/billing](https://docs.useplunk.com/concepts/billing)："Free tier projects can send up to 1,000 emails per month at no cost. Free-tier projects include a Plunk-branded footer in marketing emails. Inbound emails count toward the 1,000-email allowance... After upgrading... $0.001 per email."
- [/concepts/billing](https://docs.useplunk.com/concepts/billing)："Activating a subscription places two charges of 1.00 on your card... **Both are credited back to your account balance in full**... If the verification charge is declined, the second charge never completes, the onboarding fee is refunded in full, and your subscription is cancelled."
- [/concepts/billing](https://docs.useplunk.com/concepts/billing)："Transactional / Campaigns / Workflows / Inbound... Configure caps in **Settings → Billing → Limits**. Each category cap is independent — you can set just one, all of them, or none."
- [/concepts/billing](https://docs.useplunk.com/concepts/billing)："**Outbound API calls** (`/v1/send`, campaign sends, workflow sends) return `402 Payment Required` with error code `BILLING_LIMIT_EXCEEDED`... **Inbound emails** are dropped silently for the affected project until the cap resets. They aren't queued or replayed. Limits reset at the start of each billing period."
- [/self-hosting/environment-variables](https://docs.useplunk.com/self-hosting/environment-variables)："Required if you want to enable billing features. All Stripe variables must be set together for billing to activate."（`STRIPE_SK` / `STRIPE_WEBHOOK_SECRET` / `STRIPE_PRICE_ONBOARDING` / `STRIPE_PRICE_EMAIL_USAGE` / `STRIPE_METER_EVENT_NAME`，默认 `emails`）
- [/self-hosting/environment-variables](https://docs.useplunk.com/self-hosting/environment-variables)："If you use a Stripe Radar rule to request 3D Secure, exclude off-session payments from it (`and not :is_off_session:`). The verification charge runs with no cardholder present, so a 3DS request it cannot satisfy will leave every verification unresolved."

## I 解释

Plunk 的计费模型只有一句话：**出站按收件人算、入站按封数算，带附件翻倍**。它不区分"营销"和"事务"的价格，两类共用同一个月度用量池——所以控制成本的位置不在通道选择，而在**上限（cap）设计与超限处理**。

第一层是计数口径。事务发送 `/v1/send` 与 campaign 都是 **1 credit/收件人**，workflow 的 `SEND_EMAIL` 步骤是 **1 credit/次发送**，入站是 **1 credit/收到一封**。附件是唯一的乘数：只要带 1 个及以上附件，整封按 **2 credits** 计，且与附件体积无关（100 KB 和 10 MB 一样）。注意入站也计费——很多人以为"收邮件不花钱"，实际上入站同样计入免费层的 1000 封额度。

| 来源 | 计费口径 |
| --- | --- |
| 事务发送 `/v1/send` | 1 credit / 收件人 |
| Campaigns | 1 credit / 收件人 |
| Workflow `SEND_EMAIL` 步骤 | 1 credit / 次发送 |
| 入站邮件 | 1 credit / 收到一封 |
| 带附件的邮件 | 2 credits / 封（与附件体积无关） |

第二层是额度与单价。免费层每月 1000 封（出站 + 入站共用），代价是营销邮件强制带 Plunk 品牌页脚；升级后按量计费 **$0.001/封**，无月费无承诺。价格只适用于 useplunk.com 托管版，自托管跑在自己的基础设施上。

第三层是四类上限的分类口径与超限行为：

| 类别 | cap 管的是 | 超限后 |
| --- | --- | --- |
| Transactional | 经 `/v1/send` 发出的邮件 | 402 `BILLING_LIMIT_EXCEEDED` |
| Campaigns | 作为 campaign 一部分发出的邮件 | 402 `BILLING_LIMIT_EXCEEDED` |
| Workflows | workflow 内 `SEND_EMAIL` 步骤发出的邮件 | 402 `BILLING_LIMIT_EXCEEDED` |
| Inbound | 发到已验证域名的入站邮件 | 静默丢弃，不排队不重放 |

这四类的 cap **相互独立、可任意组合**（只设一个、全设、或全不设）。典型动机是隔离：即使某次 campaign 误发，事务邮件的容量也不会被吃光。这是把"成本失控"变成"可控降级"的唯一手段。

第四层——也是本卡最需要警示的一点——**超限后两侧行为不对称**。出站（`/v1/send`、campaign、workflow）会显式返回 `402 Payment Required`，错误码 `BILLING_LIMIT_EXCEEDED`，应用可以排队或向用户报错；而**入站邮件被静默丢弃，不排队、不重放**，一直丢到 cap 重置为止。也就是说入站超限是一条**无声的数据丢失通道**：没有任何错误码可以捕获它，只能靠通知与监控发现。限额在每个账期开始时重置，接近或触顶时会有邮件/站内通知。

第五层是订阅激活的双扣款。激活时会有两笔 1.00：onboarding fee（结账单次，支持开发）与 card verification（离场扣款能力验证）。**两笔都全额退回账户余额**，并在开票前抵扣用量，所以净成本为零。第二笔存在的原因是它测的是"你不在场时"的自动扣款——预付卡、虚拟卡、一次性卡常常第一笔过、第二笔拒。验证被拒 → 第二笔不发生、onboarding 全额退款、订阅取消。

第六层是自托管的开关前提。计费功能要五个 Stripe 变量**齐全才激活**（全有或全无），且若开了 Radar 的 3DS 规则，必须在规则里排除 off-session 支付，否则离场的验证扣款永远无法完成。

## A1 示例

**示例 1 · 带附件发票的用量估算**（合成演练，按 `/concepts/billing` 计数口径可复算）

- 输入/问题：每月给 5000 个客户各发一封带 PDF 发票的事务邮件，按"一封一 credit"估算是 5000 credits。
- 怎么执行：按口径重算——带 1 个及以上附件 → 整封 2 credits，与附件大小无关；附件上限为 10 个 / 10 MB（SES 总消息 ≤ 40 MB）。
- 产出/预期输出：实际 10000 credits、约 $10（$0.001/封）。撞上 cap 的时间点比预估早一倍。⚠️ 文档未说明"2 credits per email"是否会与收件人数相乘，多收件人场景需自行验证。

**示例 2 · 用分类 cap 保住事务容量**（合成演练，按 `/concepts/billing` 的 cap 设计可复算）

- 输入/问题：担心一次误发的 marketing campaign 把当月额度吃光，导致注册验证码发不出去。
- 怎么执行：在 Settings → Billing → Limits 给 Campaigns 设 cap（四类独立，可只设这一个），事务类不设或设更高。
- 产出/预期输出：campaign 触顶后发信返回 `402` + `BILLING_LIMIT_EXCEEDED`，应用侧排队或显式报错；事务通道不受影响。文档未给出部分发送（部分成功）的行为说明。

**示例 3 · 入站超限是无声的**（合成演练，按 `/concepts/billing` 的超限分支）

- 输入/问题：给 Inbound 设了 cap，业务侧只处理了 HTTP 402。
- 怎么执行：入站超限不发任何错误——邮件被静默丢弃，不排队也不重放，直到账期开始才恢复。
- 产出/预期输出：出站侧有 402 可捕获；入站侧**零信号**，只能靠 cap 接近/触顶的通知与自建监控发现。已丢的邮件无法找回。

**示例 4 · Radar 3DS 规则让验证永远挂起**（来自 `/self-hosting/environment-variables` 的 worked example）

- 输入/问题：自托管实例配齐了 Stripe 变量，但用户的订阅激活总是卡住。
- 怎么执行：检查 Stripe Radar 是否配了"请求 3D Secure"的规则；激活的第二笔验证扣款是 off-session（无持卡人在场），无法满足 3DS → 每一次验证都无法完成。修法是在规则里排除离场支付：`and not :is_off_session:`。
- 产出/预期输出：加上例外后验证可完成。⚠️ 文档说明：不加该例外会导致每一次验证都无法完成；验证扣款被拒则第二笔不发生、onboarding 全额退款、订阅取消。

## A2 何时会用

**用户场景**

1. 估算月度账单，尤其是批量发送带附件（发票、凭证、报告）的邮件。
2. 设计分类 cap，要保证事务邮件容量不被 campaign 吃掉。
3. 发送突然失败或收不到入站邮件，怀疑撞上了限额。
4. 订阅激活时看到卡上出现两笔 1.00 扣款，或被通知验证失败、订阅被取消。
5. 自托管要开启计费，需要知道 Stripe 变量与 Radar 规则的坑。

**语言信号（中英双写）**

`计费` / `billing` / `credits` / `1 credit per recipient` / `2 credits` / `附件` / `attachment` / `入站计费` / `inbound emails count` / `免费层` / `free tier` / `1000 封` / `$0.001` / `上限` / `cap` / `billing limits` / `402` / `BILLING_LIMIT_EXCEEDED` / `静默丢弃` / `dropped silently` / `Stripe` / `Radar` / `3DS` / `off-session`

**一句话判据**：出站超限你能看见（402），入站超限你看不见（静默丢弃）——成本可控的唯一手段是提前设 cap，而不是等超限再处理。

**与相邻能力的区分**

- 本卡负责：计费计数口径、额度与单价、四类 cap 的设计、超限后的两侧行为、订阅激活与 Stripe 配置。
- `plunk-transactional-send` 负责 `/v1/send` 的请求字段与响应；本卡只负责它消耗多少 credit 与超限时的 402。
- `plunk-inbound-email` 负责入站解析、verdict 与 `email.received` 事件；本卡只负责入站计费与超限丢弃这一条。
- `plunk-campaigns-analytics` 与 `plunk-workflows` 负责各自通道的用法；本卡只负责它们的 credit 归属类别（Campaigns / Workflows）。
- `plunk-self-hosting` 负责 Stripe 环境变量所在的部署配置总表；本卡只取"全有或全无 + Radar 例外"这两条计费相关规则。
- `plunk-api-essentials` 负责 402 之外的错误码体系与重试；本卡只针对 `BILLING_LIMIT_EXCEEDED` 这一条。

## E 执行步骤

**输入契约**

必需信息：
1. 各通道的月度量级：事务、campaign、workflow、入站各多少，其中带附件多少。
2. 当前计划：免费层还是按量；是否已设 cap、设在哪几类。
3. 遇到的症状：出站报错（含错误码）还是入站收不到。
4. 是否自托管（决定 Stripe 变量与 Radar 规则是否相关）；订阅激活场景还需扣款与通知现状。

缺失时先问：带不带附件、入站量有多大、报错的错误码是不是 `BILLING_LIMIT_EXCEEDED`。

**输出契约**

产出：月度 credit 估算（区分带附件与不带）、四类 cap 的配置建议（含隔离动机）、以及超限处理方案（出站 402 分支 + 入站监控告警）。验收：① 估算能说清每类的 credit 数与假设；② cap 配置能解释"为什么这么设"；③ 出站有 402 处理分支，入站**有非 402 的发现手段**（通知或自建监控）。

1. **先算清 credit**：事务与 campaign 按收件人 × 1、workflow 按 SEND_EMAIL 次数 × 1、入站按收到封数 × 1；带 1 个及以上附件整封 × 2（与附件大小无关；上限 10 个附件 / 10 MB，总消息 ≤ 40 MB）。完成标准：给出一个按类别拆开的月度 credit 数，并写下所依据的假设。
2. **确认额度与单价**：免费层 1000 封/月（出站 + 入站共用，营销邮件带 Plunk 品牌页脚）；升级后 $0.001/封、无月费无承诺，价格仅适用托管版。完成标准：能说出当前计划下"还剩多少额度"或"预计多少钱"。
3. **按隔离动机设 cap**：在 Settings → Billing → Limits 为 Transactional / Campaigns / Workflows / Inbound 设月度上限；四类独立，可只设一个、全设或全不设。典型用法是给 Campaigns 设 cap 以保住事务容量。完成标准：每一个已设的 cap 都能说出它保护的是哪一类发送。
4. **出站侧实现 402 分支**：捕获 `402` + `BILLING_LIMIT_EXCEEDED`，做法是排队延后或向用户显式报错，不要当成 5xx 盲目重试（重试只会继续撞墙）。完成标准：代码里有针对该错误码的分支，且不会无限重试。
5. **入站侧必须自建发现手段**：入站超限是静默丢弃，不排队、不重放，没有任何错误码。依赖"接近/触顶"的邮件与站内通知 + 自建监控（对比预期入站量与实际 `email.received` 数）。完成标准：存在一条不依赖 402 的入站丢失告警路径。
6. **说明恢复时点**：限额在每个账期开始时重置，被丢弃的入站邮件不会补发。完成标准：干系人知道"等到账期开始才恢复，且期间数据已永久丢失"。
7. **订阅激活的双扣款要提前告知**：激活会产生两笔 1.00（onboarding + 卡验证），**两笔都全额退回余额**并在开票前抵扣，净成本为零；验证被拒则第二笔不发生、onboarding 全额退款、订阅取消并邮件通知。完成标准：用户看到卡上两笔 1.00 时不会被当成误扣，且知道应换一张支持循环扣款的卡（预付卡/虚拟卡/一次性卡最易失败）。
8. **自托管开启计费要配齐 Stripe 变量**：`STRIPE_SK`、`STRIPE_WEBHOOK_SECRET`、`STRIPE_PRICE_ONBOARDING`、`STRIPE_PRICE_EMAIL_USAGE`、`STRIPE_METER_EVENT_NAME`（默认 `emails`）**全部设置**才会激活（全有或全无）。完成标准：五个变量齐全，缺任一则计费不生效。
9. **Radar 规则加 off-session 例外**：若用 Stripe Radar 请求 3D Secure，规则里必须加 `and not :is_off_session:`，否则第 7 步的离场验证扣款永远无法完成。完成标准：3DS 规则已排除离场支付。
    - 若症状是"发信失败且返回 402"：走第 4 步，并回到第 3 步检查 cap 设置是否合理。
    - 若症状是"入站邮件不见了但没有任何报错"：走第 5–6 步，先确认是否撞了 Inbound cap。
    - 若场景是订阅激活失败：走第 7 步；自托管再叠加第 8–9 步。
    - ⚠️ 文档口径不一致，两者并列，不要替文档做取舍：① 计数口径——文档同时出现 "2 credits per email" 与 "1 credit per recipient" 两种措辞，未说明带附件时是否再乘以收件人数，也未说明多收件人 `/v1/send` 下"每收件人"是否等价于收件人数组长度、测试发送与重复发送是否计费；预算请按保守值（可能翻倍）估算并自行验证。② 免费层超额行为——文档只给出 $0.001/封的按量单价与 1000 封免费额度，未说明超出 1000 封后是拒绝发送（402）还是自动转为按量计费。③ 订阅激活的双扣款与 Radar 3DS 冲突：`/concepts/billing` 把它描述为 Plunk 的正常流程，`/self-hosting/environment-variables` 把它描述为 Stripe 侧的交互约束——这可能是 Stripe 侧的交互缺陷而非 Plunk 行为，操作时需人工确认。

## B 边界与失效

1. **不要只处理 402**：这是最危险的失效模式。入站超限完全无声（静默丢弃、不排队、不重放），只实现 402 分支等于对一半的超限场景毫无感知。
2. **不要按"一封一 credit"估算带附件的量**：带附件翻倍且与体积无关，批量发票类场景最容易算错并提前撞 cap。
3. **入站不是免费的**：入站计入月度用量，也占免费层 1000 封额度；开了入站又低估入站量会直接吃掉出站预算。
4. **不要盲目重试 402**：`BILLING_LIMIT_EXCEEDED` 不是瞬时故障，重试只会在账期结束前持续失败；正确做法是排队或显式报错。
5. **附件有硬上限**：10 个附件 / 10 MB（SES 总消息 40 MB），超限会在发送侧失败；不要把"带附件翻倍"当作可以无限加附件的许可。
6. **被丢弃的入站邮件不可恢复**：不排队不重放，账期开始才恢复；这不是"延迟投递"，是永久丢失。
7. **自托管计费是全有或全无**：五个 Stripe 变量缺一个就不激活；不要假设"先配一部分也能跑"。
8. **免费层超额行为未定义**：文档未说明超 1000 封后是拒绝还是自动按量，不要按任一假设做容量规划。
9. **本卡不负责单通道的用法**：`/v1/send` 字段见 `plunk-transactional-send`，入站解析见 `plunk-inbound-email`，Stripe 之外的部署配置见 `plunk-self-hosting`。
10. **高危/不可逆操作**：入站超限期间的邮件丢弃（不可恢复，只能靠提高 cap 或降低入站量规避）；订阅验证失败导致的订阅取消（需换支持循环扣款的卡重新激活，onboarding 费会全额退款）。
