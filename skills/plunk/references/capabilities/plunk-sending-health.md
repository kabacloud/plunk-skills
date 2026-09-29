# 退信、投诉与列表卫生

## R 原文依据

- [/guides/list-hygiene](https://docs.useplunk.com/guides/list-hygiene)："A bounce occurs when an email cannot be delivered to the recipient's inbox. Plunk records every bounce as an `email.bounce` event on the contact, and automatically unsubscribes the contact **only on permanent (hard) bounces**."
- [/guides/list-hygiene](https://docs.useplunk.com/guides/list-hygiene)："Permanent (hard)... Auto-unsubscribes? Yes. Transient (soft)... No. Undetermined... No."
- [/guides/list-hygiene](https://docs.useplunk.com/guides/list-hygiene)："When Plunk receives that complaint, the contact is **always automatically unsubscribed** and Plunk fires an `email.complaint` event on the contact... Complaints are taken more seriously than bounces by mail providers."
- [/guides/list-hygiene](https://docs.useplunk.com/guides/list-hygiene)："Only Permanent bounces count toward your project's bounce rate and trigger automatic contact unsubscription. Transient bounces are tracked for visibility only."
- [/guides/list-hygiene](https://docs.useplunk.com/guides/list-hygiene)："To rehearse your bounce and complaint handling, send to the Amazon SES mailbox simulator instead of to a real address. Mail sent to `@simulator.amazonses.com` never leaves AWS... What they do **not** do is count toward your bounce and complaint rates."
- [/guides/list-hygiene](https://docs.useplunk.com/guides/list-hygiene)："The most expensive complaint is the one from someone who wanted less email, not none. If unsubscribing is the only option on the page, a recipient who is merely overwhelmed either leaves permanently or — worse for your reputation — hits the spam button instead."
- [/guides/list-hygiene](https://docs.useplunk.com/guides/list-hygiene)："Plunk's hosted unsubscribe and preferences pages offer **snoozing** alongside unsubscribing: the recipient stops receiving email for 2 weeks, 1 month, 6 months, or a year, and is resubscribed automatically when the window ends."
- [/guides/list-hygiene](https://docs.useplunk.com/guides/list-hygiene)："Every email Plunk sends carries two versions of your content: the HTML you wrote, and a plain text version generated from it... An HTML-only message is a long-standing spam signal."

## I 解释

Plunk 会自动监测退信率与投诉率，因为这两个指标直接决定发信声誉——过高会伤害可投递性，甚至导致项目被停用。所以这一卡关心的不是"某封邮件有没有送到"，而是"整个发信域的信誉是否被消耗"。

第一层是退信的三级分类，且只有一级会改变订阅状态。**Permanent（硬退）**：地址无效、域名被封、收件人被拒 → 自动退订该联系人，并计入项目退信率。**Transient（软退）**：邮箱满、服务器不可用、greylisted → 只记录 `email.bounce` 事件做可见性，不改订阅状态（通常上游会重试）。**Undetermined**：上游无法归类 → 也不改。两类都会发 `email.bounce` 事件，靠载荷的 `bounceType` 字段区分；webhook 侧软退还带 `transientBounce: true` 且不写 `bouncedAt`。因此"收到 bounce 事件就退订"是明确错误的处理方式。

| 分类 | 典型原因 | 自动退订？ | 计入项目退信率？ | 该怎么处理 |
| --- | --- | --- | --- | --- |
| Permanent（硬退） | 地址无效、域名被封、收件人被拒 | 是 | 是 | 接受自动退订；进入抑制名单，不再发送 |
| Transient（软退） | 邮箱满、服务器不可用、greylisted | 否 | 否 | 仅可见性监控，通常上游会重试 |
| Undetermined | 上游无法归类 | 否 | 否 | 仅可见性监控，不要据此改状态 |
| Complaint（投诉） | 收件人标记为垃圾邮件 | 是（一律） | 计入投诉率 | 最严重：按绝对数盯，约 0.1% 即风险 |

第二层是投诉，它比退信更严重且没有分级。收件人在邮箱服务商里标记为垃圾邮件时，Plunk 收到投诉后**一律**自动退订该联系人并打 `email.complaint` 事件。严重性来自计量尺度：服务商按千分之几计量，持续高于约 0.1% 就会危及发信能力——这就是为什么"几起投诉"也必须当信号处理，也是为什么 dashboard 的投诉卡用绝对数领头。硬退与投诉导致的自动退订会在 `contact.unsubscribed` 事件上带 `reason: "bounce" | "complaint"`，可用于区分来源。

第三层是演练方法。要验证 bounce / complaint 的处理链路，应该发给 SES 邮箱模拟器而不是真实地址：`bounce@`（硬退）、`complaint@`（投诉）、`success@`（成功投递）、`ooto@`（休假自动回复）、`suppressionlist@`（如同在抑制名单上的硬退）。这些邮件不出 AWS，但在 Plunk 里完全可见——事件会触发、workflow 与 webhook 会跑、联系人动态会显示；唯一的差别是**不计入退信率与投诉率**，因此永远不会触发警告或项目停用。

第四层是"减量而不是退出"的产品原则。最昂贵的投诉来自"只是想少收一点"的人：如果页面上只有退订这一个选项，被邮件淹没的收件人要么永久离开，要么——对声誉更糟——去点垃圾邮件按钮。托管退订页与偏好页因此同时提供 snooze：2 周 / 1 个月 / 6 个月 / 1 年，窗口期内收不到邮件，到期自动重新订阅。它是真实的 opt-out（整个窗口 `subscribed: false`），所以既保护收件人收件箱也保护你的投诉率；同时 snooze 仍计入 campaign 的退订数，因此它不是掩盖问题的手段。

第五层是后果的落点：硬退与投诉都会**抑制**该联系人（不只是翻 `subscribed`），因此被抑制的地址不会被随后到期的 snooze 复活；自动退订时 `contact.unsubscribed` 事件会带 `reason: "bounce" | "complaint"`，所以同步类 webhook 可以用 `event.reason ?? 'user_action'` 区分"用户主动退订"与"系统抑制"。这也意味着：看退订量时要把这两类分开统计，否则会把"地址质量差"误读成"内容不受欢迎"。

第六层是纯文本替代件的投递意义。每封邮件都同时带 HTML 与由 HTML 自动生成的 text 版本（事务、活动、测试发送都一样，API 没有 plain text 字段）。原因很直接：HTML-only 是长期垃圾信号（SpamAssassin 的 `MIME_HTML_ONLY` / `MPART_ALT_DIFF`），干净的邮件也可能仅靠累计分进垃圾箱。

## A1 示例

**示例 1 · 硬退与软退的不同处理**（来自 `/guides/list-hygiene` 的 bounce 表）

- 输入/问题：一次发送后出现 30 个 `email.bounce` 事件，团队决定"全部退订"。
- 怎么执行：按载荷的 `bounceType` 分支——只有 `Permanent` 应改订阅状态（Plunk 会自动做，并计入退信率）；`Transient`（邮箱满 / 服务器不可用 / greylisted）与 `Undetermined` 不应改状态。
- 产出/预期输出：错误做法会让本该继续接收的人被误退订；正确做法是只对 Permanent 做抑制动作，其余仅做监控。

**示例 2 · 安全演练投诉处理链路**（来自 `/guides/list-hygiene` 的 worked example）

- 输入/问题：想在上线前确认"投诉 → 自动退订 → webhook 回写自家库"这条链路通不通，又不想真的伤害声誉。
- 怎么执行：往 `complaint@simulator.amazonses.com` 发一封（硬退用 `bounce@`，成功用 `success@`，自动回复用 `ooto@`，抑制名单场景用 `suppressionlist@`）。
- 产出/预期输出：Plunk 侧表现与真实投诉完全一致——`email.complaint` 触发、联系人被退订、workflow 与 webhook 执行、动态流可查；但**不计入**投诉率与退信率，也不会触发警告或项目暂停（AWS 侧同样排除）。

**示例 3 · 用 snooze 接住"想少收"的人**（来自 `/guides/list-hygiene` 与 `plunk-subscription-consent`）

- 输入/问题：退订页只有"退订"按钮，最近投诉数上升。
- 怎么执行：改用托管的 unsubscribe / preferences 页（自带 snooze：2 周 / 1 个月 / 6 个月 / 1 年，到期自动重新订阅），或自建 UI 复用免鉴权端点 `POST /contacts/public/:id/snooze?e=<emailId>`，body `{ duration: "6_months" }`。
- 产出/预期输出：想减量的人转为 snooze 而非永久离开或点垃圾邮件；snooze 仍计入 campaign 退订数，所以不会掩盖"这次活动在消耗好感"。注意：硬退或投诉会抑制该地址，被抑制的联系人不会被进行中的 snooze 复活。

**示例 4 · 入站前的地址质量把关**（合成演练，按 `/guides/list-hygiene` 的预防清单可复算）

- 输入/问题：注册表单直接写入的名单里混入大量无效地址，退信率上升。
- 怎么执行：三条预防动作并行——① 注册时用 `POST /v1/verify` 校验地址（格式、MX、一次性域名、拼写纠错）；② 定期清理：把不活跃联系人分群出来重新确认或移除；③ 用 workflow 做确认流（发验证邮件，只有点击通过才标记为已确认，且这封确认邮件必须用 TRANSACTIONAL 模板，否则对方退订后就收不到）。
- 产出/预期输出：硬退减少、项目退信率下降，且名单里剩下的人是经过 opt-in 的。具体改善幅度文档未说明；注意 CSV 导入的联系人默认 subscribed，会让从未 opt-in 的人直接进列表（见 `plunk-contacts-and-fields`）。

**示例 5 · 投诉升高后的三条并行动作**（合成演练，按 `/guides/list-hygiene` 的预防清单可复算）

- 输入/问题：最近两次 newsletter 的投诉数从 1–2 起涨到十几起，团队只做了"把投诉的人删掉"。
- 怎么执行：三条线并行——① 内容侧：确认每封营销邮件都有清晰可用的退订链接（MARKETING 类型自动注入；HEADLESS 必须自己放 `{{unsubscribeUrl}}` / `{{manageUrl}}`）；② 频控侧：降低发送频率、把不活跃联系人分群出来重新确认或移除；③ 入口侧：确认名单都经过 opt-in，停止使用购买/抓取/陈旧名单，并在注册时用 `POST /v1/verify` 校验。
- 产出/预期输出：投诉数回落。⚠️ 注意：删人只处理了后果，只要入口与频控不变，指标会持续被消耗；具体改善幅度文档未说明。

## A2 何时会用

**用户场景**

1. 退信率或投诉率升高，需要判断是哪一类退信、会不会自动退订、要不要人工介入。
2. 要在 workflow / webhook 里处理 `email.bounce` / `email.complaint`，不确定该怎么分支。
3. 想在上线前演练退信与投诉处理，又不能真的伤害发信声誉。
4. 退订页只有"退订"一个选项，怀疑投诉来自"只是想少收"的人。
5. 名单质量差（购买/抓取/陈旧/未经 opt-in），需要一套清理与预防动作。
6. 想确认为什么每封邮件都有一个自己没写过的纯文本版本，或担心 HTML-only 影响投递。

**语言信号（中英双写）**

`退信` / `bounce` / `hard bounce` / `soft bounce` / `Permanent` / `Transient` / `Undetermined` / `bounceType` / `email.bounce` / `投诉` / `complaint` / `email.complaint` / `spam` / `垃圾邮件` / `列表卫生` / `list hygiene` / `sender reputation` / `0.1%` / `SES simulator` / `snooze` / `plain text alternative` / `MIME_HTML_ONLY`

**一句话判据**：看到 `email.bounce` 先问 `bounceType`（只有 Permanent 会改状态），看到 `email.complaint` 直接按最严重处理——两者都不是"删掉这条记录"就能解决的信号，背后是名单与内容问题。

**与相邻能力的区分**

- 本卡负责：退信与投诉的分级、自动退订后果、声誉风险的日常监测与列表卫生动作。
- `plunk-subscription-consent` 负责订阅状态、退订/订阅/偏好页与 snooze 的完整语义；本卡只取"snooze 用来压投诉率"这一条。
- `plunk-campaigns-analytics` 负责单次活动的统计口径（三列互斥、投诉率怎么读）；本卡负责读到异常之后的卫生动作。
- `plunk-webhooks` 负责 `email.bounce` / `email.complaint` 的出站 payload 字段与重试；本卡只关心用哪个字段分支。
- `plunk-domain-deliverability` 负责域名与 DNS 认证（SPF/DKIM/DMARC）；本卡处理的是"域名已验证但声誉在变差"的那一层。
- `plunk-templates-liquid` 负责模板内容与自动生成的纯文本保真规则；本卡只解释为什么要保留纯文本版本。

**按诉求快速分流**：

- "退信/投诉怎么处理、声誉怎么救" → 本卡。
- "退订页、托管页、snooze 的完整语义" → `plunk-subscription-consent`。
- "这次活动的三列统计怎么读" → `plunk-campaigns-analytics`。
- "webhook 载荷字段与重试" → `plunk-webhooks`。
- "SPF/DKIM/DMARC 与域名验证" → `plunk-domain-deliverability`。

## E 执行步骤

**输入契约**

必需信息：
1. 观察到的信号：是 `email.bounce`（含 `bounceType`）还是 `email.complaint`，以及涉及多少联系人。
2. 当前项目退信率与投诉率（投诉按绝对数与百分比一起看）。
3. 名单来源与 opt-in 方式（是否有购买/抓取/未经确认的地址）。
4. 退订页现状：是否提供 snooze 这类"减量"选项。
5. 是否处于自托管环境（决定 `AUTO_PROJECT_DISABLE` 是否可能自动停用项目）。

缺失时先问：是否已在监听 `email.complaint`（未监听等于毫无感知）、营销邮件是否每封都有可用退订链接、名单是否经过确认流。

**输出契约**

产出：分级处置结论（哪些联系人被自动退订、哪些只是可见性记录）、演练方案（模拟器地址与预期表现）、以及一份列表卫生动作清单（校验、清理、确认流、频控、snooze）。验收：① 每个 `email.bounce` 都能说出 `bounceType` 与对应动作；② 投诉处理链路在模拟器上跑通且未污染真实指标；③ 预防动作有明确责任人与周期。

1. **先分清是退信还是投诉**：`email.bounce` 走第 2 步，`email.complaint` 走第 3 步。完成标准：事件名与处理路径一一对应，不混用。
2. **退信按三级分支**：读事件载荷的 `bounceType` —— `Permanent` 才算硬退（Plunk 已自动退订并计入项目退信率），`Transient` / `Undetermined` 只做可见性监控、不改订阅状态（webhook 侧软退表现为 `transientBounce: true` 且不写 `bouncedAt`）。完成标准：下游系统不会把软退误写成退订，也不会把硬退当成"下次再试"。
3. **投诉一律按最严重处理**：收到投诉即自动退订并触发 `email.complaint`；用 `contact.unsubscribed` 事件上的 `reason`（`bounce` / `complaint`）区分来源，或在 webhook 里用 `event.reason ?? 'user_action'` 判断。完成标准：投诉数按绝对数记录并与 0.1% 警戒线比较——哪怕只有几起也要立案。
4. **未监听就先补监听**：建 workflow 监听 `email.complaint` 并 webhook 回自己的库，否则联系人被退订、投诉率上升时业务侧毫无感知。完成标准：投诉发生后自家系统能看到记录与告警。
5. **上线前用模拟器演练**：往 `bounce@` / `complaint@` / `success@` / `ooto@` / `suppressionlist@`（`@simulator.amazonses.com`）发送，核对事件、workflow、webhook 与动态流。完成标准：链路全通，且确认这些发送**不计入**退信率与投诉率、不会触发警告或项目停用。
6. **给"想少收"的人一条退路**：退订页不要只有退订——启用托管页自带的 snooze（2 周 / 1 个月 / 6 个月 / 1 年，到期自动重新订阅），或自建 UI 复用 `POST /contacts/public/:id/snooze?e=<emailId>`。完成标准：退订页上同时存在"暂停"与"退订"两个选项，且知道 snooze 仍计入 campaign 退订数（不是掩盖手段）。
7. **入口把关地址质量**：注册时用 `POST /v1/verify` 校验（格式、MX、一次性域名、拼写纠错），不要直接写入未校验地址。完成标准：新入库地址都经过校验，拒绝项有明确处理（提示重填而不是静默丢弃）。
8. **定期清理不活跃名单**：把长期无互动的联系人分群出来，重新确认或移除。完成标准：有固定的清理周期与判定条件，而不是等到退信率升高才动。
9. **用确认流替代直接入库**：发验证邮件，只有点击通过才标记为已确认（double opt-in 的确认邮件必须用 TRANSACTIONAL 模板，否则对方退订后就收不到）。完成标准：未经确认的地址不进入常规营销发送。
10. **控频与内容相关性**：监控发送频率避免淹没收件人；确保内容相关且有价值；确保每封营销邮件有清晰可用的退订链接（MARKETING 类型由 Plunk 自动注入，HEADLESS 必须自己放 `{{unsubscribeUrl}}` / `{{manageUrl}}`）。完成标准：营销类邮件的退订链接逐封可见可点。
11. **不要碰纯文本这一侧**：每封邮件都会自动生成 text 替代件，API 没有 plain text 字段；不要试图塞入与 HTML 不同的内容（不一致会触发它本想规避的同一批过滤器）。完全无可读文本时 Plunk 会直接省略 text 部分。完成标准：接受自动生成结果，把精力放在 HTML 的可读性上。
12. **把监测做成例行而不是事后**：固定节奏看三件事——项目退信率与投诉率（投诉按绝对数 + 百分比）、`email.complaint` 的告警是否到达、不活跃名单的规模。完成标准：有明确的周期与阈值（投诉率持续高于约 0.1% 即立案），且每一条告警都能落到第 7–10 步的某个动作上。
    - 若事件是 `email.bounce`：先读 `bounceType`，`Permanent` 才按抑制处理（回到第 2 步），其余只记录监控。
    - 若事件是 `email.complaint`：直接进入第 3–4 步，并按最严重级别立案。
    - 若只是要验证链路：走第 5 步的模拟器，不要拿真实名单做演练。
    - 若自托管且项目被自动停用：先查 `AUTO_PROJECT_DISABLE` 与退信/投诉率，修好名单与内容再恢复，不要只停用开关。
    - ⚠️ 文档口径不一致，两者并列，不要替文档做取舍：campaign 概念页把投诉阈值表述为"持续高于约 0.1% 会危及发信"，并把 snooze 计入 campaign 退订数；而活动统计页对 `unsubscribedCount` 的归因限定为"可归因的主动退订"。统计投诉与退订时按两种口径分别记录，不要自行合并。

## B 边界与失效

1. **不要用"有没有 bounce 事件"代替分级**：软退与 Undetermined 都不改订阅状态，只做可见性记录；把它们一律当硬退处理会误退订真实用户。
2. **投诉没有软档也没有豁免**：收到即自动退订，且比退信严重得多（按千分之几计量，约 0.1% 是持续值警戒线）。不要因为"只有几起"就跳过处理。
3. **未监听等于未处理**：`email.complaint` 触发的自动退订是静默发生的，没有 workflow/webhook 监听时业务侧完全无感知，声誉已在恶化。
4. **模拟器只能验证链路，不能验证规模**：模拟器地址不计入退信率与投诉率（也不触发警告/停用），所以它无法告诉你真实阈值下的表现；真实规模的风险要靠生产指标观察。
5. **snooze 不是万能药**：它是真实的 opt-out（窗口内 `subscribed: false`），且仍计入 campaign 退订数；被硬退或投诉抑制的地址不会被进行中的 snooze 复活。
6. **列表卫生不能靠一次清理解决**：购买、抓取、陈旧、未经 opt-in 的名单会从入口持续污染指标，必须入口校验 + 定期清理 + 确认流三件事一起做。
7. **纯文本不可控**：自动生成且没有字段可设，无可读文本时会被直接省略；不要在这一侧设计内容或做 A/B。
8. **本卡不覆盖 DNS 与认证问题**：SPF/DKIM/DMARC 与域名验证属于 `plunk-domain-deliverability`；本卡处理的是"认证没问题但声誉在变差"。
9. **自托管要知道会停项目**：`AUTO_PROJECT_DISABLE`（默认 true）会在退信/投诉率超阈值时自动暂停项目——演练请走模拟器，生产请盯住这两个率，恢复前先解决名单与内容问题（见 `plunk-self-hosting`）。
10. **高危/不可逆操作**：硬退与投诉导致的自动退订会直接改变联系人状态（可通过 `PATCH` 重新订阅，但会清掉进行中的 snooze 且可能再次触发风险）；项目被自动停用会影响全部发送能力。恢复动作只能从"降率"入手，没有一键重置声誉的办法。
