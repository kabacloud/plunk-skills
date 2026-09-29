# 订阅状态、退订与合规

## R 原文依据

- [/concepts/contacts](https://docs.useplunk.com/concepts/contacts)：投递矩阵 —— 「**Transactional** (via /v1/send): Delivered / Delivered. **Campaigns** (marketing): Delivered / Not delivered. **Campaigns** (headless): Delivered / Not delivered. **Campaigns** (transactional): Delivered / Delivered. **Automations** (marketing template): Delivered / Not delivered.」
- [/concepts/contacts](https://docs.useplunk.com/concepts/contacts)：「Even when using the transactional API endpoint (`/v1/send`), you cannot send a marketing template to an unsubscribed contact. Use a transactional template instead if the email must reach unsubscribed contacts.」；「omitting `subscribed` keeps the current state — it is not the same as passing `false`.」
- [/concepts/contacts](https://docs.useplunk.com/concepts/contacts)：退订途径 —— 「Automatically on a permanent (hard) email bounce. Soft bounces don't change subscription state. Automatically when a recipient marks one of your emails as spam.」；「A snoozed contact has `subscribed: false` — so every rule on this page applies to them unchanged — plus a `snoozedUntil` timestamp saying when they come back.」
- [/guides/unsubscribe-pages](https://docs.useplunk.com/guides/unsubscribe-pages)：三个 URL 变量 —— 「`{{unsubscribeUrl}}` — One-click unsubscribe page. `{{subscribeUrl}}` — Resubscribe page. `{{manageUrl}}` — Preferences page: View their current state and toggle subscription either way.」；自动页脚「**Headless**: No — You must include `{{unsubscribeUrl}}` (or `{{manageUrl}}`) in your body or the recipient has no way to opt out.」
- [/guides/unsubscribe-pages](https://docs.useplunk.com/guides/unsubscribe-pages)：snooze —— 「2 weeks → `2_weeks`; 1 month → `1_month`; 6 months → `6_months`; 1 year → `1_year`」；「A snoozed contact has `subscribed: false` for the whole window... Nothing is "held back and delivered later".」；「Any other change to a contact's subscription cancels the snooze and takes over — the pending return is dropped, not queued behind it.」
- [/guides/unsubscribe-pages](https://docs.useplunk.com/guides/unsubscribe-pages)：归因 —— 「The links Plunk generates at send time carry an `?e=<emailId>` parameter... A few cases are unattributed by design: Links rendered before the email row exists... Changes with no email behind them... Mail sent before this parameter existed.」；「The value is advisory — Plunk verifies the email belongs to the contact before recording it, and the contact id in the path is what governs the change.」
- [/guides/list-hygiene](https://docs.useplunk.com/guides/list-hygiene)：「The most expensive complaint is the one from someone who wanted less email, not none. If unsubscribing is the only option on the page, a recipient who is merely overwhelmed either leaves permanently or — worse for your reputation — hits the spam button instead.」；「Snoozes count toward a campaign's unsubscribe rate, so this does not hide a campaign that is burning through goodwill.」

## I 解释

订阅状态在 Plunk 里是**硬约束**，而且是投递判定的唯一依据——不看走哪条通道，只看模板类型。MARKETING 与 HEADLESS 一律尊重 opt-out（已退订就不发），TRANSACTIONAL 一律绕开订阅状态投递。这条规则在 campaign、automation、以及事务端点 `/v1/send` 上完全一致，所以"改成用 `/v1/send` 发"并不能绕过退订过滤；要触达已退订的人，唯一做法是改用 TRANSACTIONAL 模板。这是排查"邮件发出去了但对方没收到"的第一判据。

第二条是状态变更的可观测性。`subscribed` 每次翻转都会自动打事件（true → `contact.subscribed`，false → `contact.unsubscribed`），而且覆盖**所有**变更路径：dashboard 手工改、托管退订页、批量接口、CSV 导入、workflow 的 UPDATE_CONTACT、以及硬退信与垃圾投诉引起的自动退订。这意味着"同步退订回自己的库"只需要监听一个事件，不必区分来源——但也意味着自动退订（bounce / complaint）会混进同一个事件流，必须在分支上读 `reason` 区分。

第三条是"暂停"（snooze）的设计意图。退订页上如果只有"永久退出"，被邮件淹没的人要么永久离开，要么去点垃圾按钮——后者对发信声誉贵得多。所以 Plunk 在退订/偏好页上提供次级选项 snooze（2 周 / 1 月 / 6 月 / 1 年）。关键在于：snooze 是**真实的 opt-out，不是更软的 opt-out**——整个窗口内 `subscribed: false`，所有发送路径都按这个标志过滤，不存在"压着以后补发"。`snoozedUntil` 只用来排回归时间，不参与是否发信的判定。

第四条是 snooze 复用既有事件名带来的代价。它不新增事件类型（好处是既有 workflow 自动兼容），代价是必须在分支上显式判断 `reason`：`snooze` 是暂停、`bounce`/`complaint` 是自动退订、无 `reason` 是手动/自助退订、`contact.subscribed` 的 `snooze_expired` 是到期回归。挽留类流程如果忘了过滤，会向只是暂停的人发"很遗憾你离开"。

第五条是归因。`List-Unsubscribe` 头、自动注入的营销页脚、以及三个 URL 变量都带 `?e=<emailId>`，收件人动作时把该邮件记到事件上，从而算出"这次 campaign 花了多少订阅者"。但有三类场景故意不可归因：队列期渲染的 campaign 正文链接、没有邮件在背后的改动（dashboard 开关 / CSV 导入 / PATCH）、以及该参数存在之前发的邮件。`?e=` 是建议值——填错最坏只丢归因，不会改错联系人。

**投递矩阵速查**

| 邮件类型 | 已订阅 | 已退订 |
| --- | --- | --- |
| Transactional（`/v1/send`） | 送达 | 送达 |
| Campaign — marketing | 送达 | **不送达** |
| Campaign — headless | 送达 | **不送达** |
| Campaign — transactional | 送达 | 送达 |
| Automation — marketing 模板 | 送达 | **不送达** |
| Automation — headless 模板 | 送达 | **不送达** |
| Automation — transactional 模板 | 送达 | 送达 |

**订阅动作 → 状态与事件映射**

| 动作 | 状态变化 | 事件 |
| --- | --- | --- |
| 退订（unsubscribe） | `subscribed` → `false` | `contact.unsubscribed`（无 `reason`） |
| 暂停（snooze） | `subscribed` → `false`，`snoozedUntil` 被设置 | `contact.unsubscribed` + `{reason:"snooze", duration, snoozedUntil}` |
| 重新订阅（resubscribe） | `subscribed` → `true` | `contact.subscribed` |
| snooze 到期自动回归 | `subscribed` → `true`，`snoozedUntil` 清空 | `contact.subscribed` + `{reason:"snooze_expired"}` |
| 硬退信自动退订 | `subscribed` → `false` | `contact.unsubscribed` + `{reason:"bounce"}`，另有 `email.bounce` |
| 垃圾投诉自动退订 | `subscribed` → `false` | `contact.unsubscribed` + `{reason:"complaint"}`，另有 `email.complaint` |

**自动页脚速查**

| 模板类型 | 自动追加含 `{{unsubscribeUrl}}` 的本地化页脚 |
| --- | --- |
| Marketing | 是（尊重收件人 `locale`） |
| Headless | 否 —— **必须自己放** `{{unsubscribeUrl}}` 或 `{{manageUrl}}` |
| Transactional | 否（跳过订阅检查，也不需要） |
| `/v1/send` 不带模板（仅 subject + body） | 否 |

## A1 示例

**示例 1：snooze 与永久退订靠 `reason` 分开**（来自 /guides/unsubscribe-pages 的 worked example）

- 输入：`contact.unsubscribed` 事件体 `{reason:"snooze", duration:"6_months", snoozedUntil:"2027-03-11T09:14:22.104Z"}`；窗口结束时 `contact.subscribed` 事件体 `{reason:"snooze_expired"}`
- 怎么执行：在触发于 `contact.unsubscribed` 的 workflow 里先读 `event.reason`——`reason == "snooze"` 就跳过挽留/告别动作
- 产出：文档说明窗口结束后后台任务在几分钟内重新订阅、清空 `snoozedUntil` 并触发 `reason:"snooze_expired"`；snooze 计入 campaign 的退订数

**示例 2：double opt-in 用 `{{subscribeUrl}}` 做确认**（来自 /recipes/double-opt-in 的 worked example）

- 输入：注册后建联系人 `{"email":"ada@example.com","subscribed":false,"data":{"firstName":"Ada"}}`，再 `POST /v1/track` 带 `{"event":"signup.pending","email":"ada@example.com","subscribed":false}`
- 怎么执行：Workflow A 监听 `signup.pending` 发 **TRANSACTIONAL** 确认模板（内含 `<a href="{{subscribeUrl}}">Confirm my email</a>`）；Workflow B 监听 `contact.subscribed` 发 MARKETING 欢迎模板
- 产出：文档说明 `contact.subscribed` 在 `{{subscribeUrl}}`、偏好页、API 任一途径 opt-in 时都会触发，因此 Workflow B 同时覆盖首次确认与重新订阅。若确认邮件误用 marketing 模板，会被静默跳过

**示例 3：把退订同步回自己的库**（来自 /recipes/sync-unsubscribes 的 worked example）

- 输入：`contact.unsubscribed` 事件
- 怎么执行：建 workflow（触发器 `EVENT` on `contact.unsubscribed` + `WEBHOOK` 步骤指向自建 HTTPS 端点）；端点校验共享 secret 后按 `contact.email` 更新用户行，写入 `event.reason ?? 'user_action'`
- 产出：文档说明 `event.reason` 为 `"bounce"` / `"complaint"` 时是自动退订，缺失时是手动/自助退订；反向用 `PATCH /contacts/:id` 改动时也会触发同一事件并回环到自己的 handler（更新幂等，通常无害）

**示例 4：三类不可归因的退订**（来自 /guides/unsubscribe-pages）

- 输入：campaign 报告的 `unsubscribedCount`
- 怎么执行：检查退订来源——正文里那条 `{{unsubscribeUrl}}` 是在入队时为每个收件人渲染一次的（那时 email 行还不存在），因此不带 `?e=`
- 产出：这类退订不计入任何 campaign 的 `unsubscribedCount`，用该指标评估"这个活动掉了多少订阅"会低估（文档未给出补偿做法，需自行验证）

## A2 何时会用

**用户场景**

1. 一封关键邮件（订单确认、密码重置）发出去了但收件人没收到，要判断是不是被订阅状态静默跳过。
2. 用 headless 模板完全自控样式，要确认退订出口放在哪、会不会因为没放链接而踩合规风险。
3. 要建"挽留/告别"自动化，需要把"只是暂停 6 个月"的人和"真的走了"的人分开。
4. 要把 Plunk 的退订实时同步回自己的用户库，并区分主动退订与硬退信/投诉引起的自动退订。
5. 用 campaign 的退订数评估活动代价，发现数字偏低，怀疑归因缺失。
6. 收件人抱怨邮件太多，需要在退订页上提供"减量"而不是只有"退出"。

**语言信号**：退订 / unsubscribe / opt-out / 订阅状态 / subscribed / 重新订阅 / resubscribe / double opt-in / 暂停 / snooze / `snoozedUntil` / `{{unsubscribeUrl}}` / `{{subscribeUrl}}` / `{{manageUrl}}` / 退订页脚 / 硬退信 / hard bounce / 垃圾投诉 / complaint / `contact.unsubscribed` / `?e=` / 归因 / unsubscribe rate

**与相邻能力的区分**

- 本卡负责：`subscribed` 的语义与投递影响、退订/重新订阅/snooze 三条路径、事件与归因、托管页与页脚的合规要求。
- `plunk-contacts-and-fields` 负责：`subscribed` 作为 contact 级保留键不能被 `data` 设置、各入口的默认值差异——本卡只在"投递判定"层面使用它。
- `plunk-templates-liquid` 负责：MARKETING / HEADLESS / TRANSACTIONAL 三种模板本身怎么建、Liquid 怎么写——本卡只负责模板类型对订阅与页脚的影响。
- `plunk-transactional-send` 负责：`/v1/send` 的请求字段与 `emailId`——本卡只负责"用 marketing 模板走 `/v1/send` 仍到不了已退订者"这一判据。
- `plunk-sending-health` 负责：退信/投诉的日常列表卫生与 SES simulator 演练——本卡只负责硬退信与投诉触发的**自动退订**这一后果。
- `plunk-campaigns-analytics` 负责：campaign 统计的完整口径——本卡只负责其中的退订计数与 snooze 计入规则。

## E 执行步骤

**输入契约**（必需信息，缺失时先问）

| 必需 | 说明 | 缺失时 |
| --- | --- | --- |
| 这封邮件是否必须到达已退订者 | 决定模板类型，是投递判据的第一问 | 先问；答"是"就必须 TRANSACTIONAL |
| 当前用的模板类型 | MARKETING / HEADLESS / TRANSACTIONAL | 先问；不知道就无法判投递 |
| 收件人的 `subscribed` 与 `snoozedUntil` | 区分永久退订与暂停 | 先回读联系人 |
| 退订出口放在哪 | headless 必须自己放链接 | 先问模板类型 |
| 是否要监听事件 | 决定是否建 workflow + webhook | 先问同步目标 |

**输出契约**

- 投递可行性的结论（能到 / 不能到 + 该换什么模板类型）
- 退订/偏好页的链接布置方案（用哪个 URL 变量、放在哪）
- snooze 与永久退订的分支判定规则（按 `event.reason`）
- 若需同步：workflow 触发器 + webhook 端点契约
- 验收方式：用 `GET /contacts/:id` 或联系人 activity feed 确认状态与事件已落地

**步骤**

1. **先问"这封邮件必须到达已退订的人吗"**：是 → 必须用 TRANSACTIONAL 模板（也只有它能绕开订阅检查）；否 → marketing / headless 并接受退订过滤。完成标准：模板类型与这个答案一致。注意：改用 `/v1/send` 通道**不能**豁免订阅检查。
2. **回读收件人状态**：取 `subscribed` 与 `snoozedUntil`。`subscribed: false` 一律不投递 marketing/headless；`snoozedUntil` 非空只说明"这是暂停，到点会自己回来"，**不参与是否发信的判定**。完成标准：两个字段都读到，且没有把 snooze 误判为可发送。
3. **确认退订出口**：MARKETING 模板由 Plunk 自动追加本地化页脚，不用管；HEADLESS **必须**自己在正文里放 `{{unsubscribeUrl}}` 或 `{{manageUrl}}`，否则收件人无法 opt-out（且这类邮件仍会被未订阅者跳过）；TRANSACTIONAL 不加也不需要。走 `/v1/send` 不带模板（仅 subject + body）时也没有自动页脚。完成标准：每封可退订邮件都至少有一个可达的退订链接。
4. **提供"暂停"而不只是"退出"**：退订页/偏好页上同时给永久退订与 snooze（2 周 / 1 月 / 6 月 / 1 年，API 取值 `2_weeks` / `1_month` / `6_months` / `1_year`）。snooze 保持次级位置，想走的人不必找按钮。完成标准：页面上两个选项都在，且 snooze 的视觉权重低于退订。
5. **程序化改状态（如需自建偏好 UI）**：`PATCH /contacts/:id` 带 `{ "subscribed": false }` 退订、`true` 重新订阅；两者都自动发对应事件，也都会清除进行中的 snooze。自建 snooze 用同一个免鉴权端点 `POST /contacts/public/:id/snooze?e=<emailId>`，body `{ "duration": "6_months" }`。完成标准：回读联系人 `subscribed` 与 `snoozedUntil` 与预期一致。注意更新联系人时**省略 `subscribed` 表示保持原状态**，不等价于传 false。
6. **建事件分支**：在监听 `contact.unsubscribed` 的 workflow 里先读 `event.reason`——`reason == "snooze"` 跳过挽留/告别；`reason == "bounce"` / `"complaint"` 走抑制与列表卫生处理；无 `reason` 走常规退订流程。另建一个监听 `contact.subscribed` 的 workflow 处理回归（含 `reason == "snooze_expired"`）。完成标准：挽留流程不会对 snooze 的人触发。
7. **同步回自己的库（可选）**：`contact.unsubscribed` → workflow `WEBHOOK` 步骤 → 自建 HTTPS 端点（校验共享 secret，10 秒内响应，慢活异步做）→ 按 `contact.email` 更新用户行并写入 `event.reason ?? 'user_action'`。完成标准：一次退订在自建库里可见，且反向用 `PATCH` 改动时回环是幂等的。
8. **处理归因**：发送时确保链接带 `?e=<emailId>`（List-Unsubscribe 头文件、自动页脚、三个 URL 变量都自带）。自建退订入口可自行追加 `?e=`——它是建议值，Plunk 会校验该 email 属于该 contact，真正决定变更的是路径里的 contact id，所以填错最坏只丢归因。完成标准：能归因的退订在活动报告里可见。
9. **核对不可归因的三类**：① 队列期渲染的 campaign 正文 `{{unsubscribeUrl}}`（email 行还不存在）；② 没有邮件在背后的改动（dashboard 开关、CSV 导入、`PATCH /contacts/:id`）；③ 该参数存在之前发的邮件。完成标准：用 `unsubscribedCount` 下结论前，已把这三类作为低估来源显式说明。
10. **核对统计口径**：snooze 与永久退订一样计入 campaign 的 unsubscribe count/rate——这是刻意设计，不让"大家选了暂停"掩盖一次消耗好感的活动。文档未说明 snooze 到期回归后是否回冲该计数。完成标准：向业务方解释退订率时已包含这一口径。
11. **排障分支**：若"邮件发出去了但对方没收到"，回到第 1 步核对模板类型；若确认是自动退订，读 `event.reason` 判断是硬退信还是投诉，转 `plunk-sending-health` 做列表卫生；若是 snooze 被意外取消，检查是否有后台同步脚本无差别写了 `subscribed`（`PATCH` / CSV 导入 / 批量操作 / workflow UPDATE_CONTACT 都会终止在途 snooze，且被抑制的地址永不被 snooze 复活）。

## B 边界与失效

1. **不要用"换通道"绕开订阅检查**：走 `/v1/send` 用 marketing 模板给已退订者发信会被静默跳过——调用方以为已发出，收件人什么都没收到。这是排查"邮件丢了"的第一判据。
2. **不要把 snooze 当"积压队列"**：窗口内完全不发，到期也不会补发期间的邮件。需要补发的场景只能自己在 workflow 里重建触发逻辑。
3. **不要无意义地带 `subscribed` 写联系人**：`PATCH /contacts/:id`、CSV 导入、批量操作、workflow 的 UPDATE_CONTACT 步骤只要改了 `subscribed`，就会**取消进行中的 snooze**（pending 回归被丢弃，不是排队延后），用户提前收到邮件，投诉风险上升。
4. **⚠️ 关于"被抑制的地址永不复活"**：文档明确"a suppressed contact is never resurrected by a snooze that was running when it happened"——snooze 期间发生硬退信或投诉，预期回归不会发生。文档未给出恢复做法，需自行验证；可确定的部分：不要依赖 snooze 的自动回归作为唯一恢复路径。
5. **headless 模板漏放退订链接是合规事故**：headless 仍尊重 opt-out（会被未订阅者跳过），但不自动加页脚。没有退订出口会推高投诉率。
6. **不要指望 `unsubscribedCount` 覆盖全部退订**：campaign 正文那条链接在入队期渲染，不带 `?e=`，因此这类退订不计入任何活动（文档未给出补偿做法）。用该指标评估活动代价会系统性低估。
7. **`?e=` 别当安全边界**：它是建议值、仅用于归因。真正决定变更的是路径里的 contact id；填错或缺失只损失归因，不会改错联系人，也不应被当作攻击向量。
8. **托管页直接用项目名自称**：收件人看到的是"Stop receiving emails from Acme at ..."，`Test` 或 `acme-prod-eu` 这类内部代号会原样展示。项目名是面向收件人的 UI 字符串；要完全控制外观就改用 headless 自建退订界面并链到 `{{unsubscribeUrl}}`。
9. **自动退订只有两类触发源**：硬退信（permanent/hard bounce）与垃圾投诉。软退信（transient）与 undetermined **不改变**订阅状态。不要把"地址暂时不可用"当成"已被退订"。
10. **snooze 计入退订率是设计而非缺陷**：不要为了减少退订数而去掉 snooze 选项——那会把本来可以暂停的人推向垃圾按钮。
11. **计数口径的未说明项**：⚠️ 文档未说明 snooze 到期回归后是否回冲 campaign 的 `unsubscribedCount`，也未说明 `?e=` 指向的邮件所属活动被删除时如何归因。这两处需自行验证，不要写成确定性结论。
12. **能力边界**：本卡不覆盖域名的 DNS 认证（`plunk-domain-deliverability`）、退信与投诉的日常监控演练（`plunk-sending-health`）、以及 webhook 的出站重试语义（`plunk-webhooks`）。

**高危不可撤销操作**

- 无差别批量改 `subscribed`：批量订阅 / CSV 导入 / `PATCH` 会一把抹掉所有进行中的 snooze 计划，让已选择暂停的用户提前收到邮件。属于"技术上成功、体验上破坏"的操作，执行前必须确认名单不含 snoozed 联系人。
- 硬退信与投诉引发的自动退订不可逆：被抑制的地址不会被任何 snooze 复活，也未被文档给出恢复路径。
