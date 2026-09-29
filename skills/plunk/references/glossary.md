# Plunk 术语表（GLOSSARY）

- 来源：https://docs.useplunk.com/ 全站 48 页，2026-09-29 抓取
- 术语总数：85。每条带 `kind`（`plunk-specific` = Plunk 专有，换产品即失效；`industry-standard` = 行业通用，只记 Plunk 侧特殊行为）与来源页
- 作用：作为 18 张能力卡的共享底座。读卡时遇到不认识的词先查这里

---

# candidates: glossary

<!-- 每条: term / kind(plunk-specific | industry-standard) / definition / source_page / aliases / related -->
<!-- kind 说明：plunk-specific = Plunk 专有，必须记住；industry-standard = 行业通用，只记 Plunk 侧特殊行为 -->

## Contact 与联系人数据

- term: contact（联系人）
  kind: industry-standard
  definition: |
    Plunk 里唯一的「一等公民」对象：代表一个个体邮件收件人，每个联系人有一个 id 并绑定一个邮箱地址。
    创建途径：`/v1/track`（事件打到不存在的邮箱会自动建联系人，默认 subscribed）、`POST /contacts`（upsert，按 email；
    已存在则更新而非报错，响应带 `_meta: { isNew, isUpdate }`，新建 201、更新 200）、CSV 导入
    （`POST /contacts/import`，≤5MB，轮询 `GET /contacts/import/:jobId`）、静态 segment 加成员时 `createMissing: true`、
    收到入站邮件时自动创建发件人、以及 dashboard 手动添加。
  source_page: https://docs.useplunk.com/concepts/contacts
  aliases: [联系人, Contact record, cnt_]
  related: [subscribed, snoozedUntil, data.* 自定义字段, reserved keys, /v1/track]

- term: data.* 自定义字段（custom fields）
  kind: plunk-specific
  definition: |
    每个联系人除内置字段（email / subscribed / createdAt / updatedAt）外都带一个 `data` 对象，可放任意 key-value，
    用于模板个性化、segment 过滤、workflow 条件。`data` 是 patch 语义：只改你传的 key。
    命名空间：`data.<path>`（segment 过滤里必须带前缀），模板里 `{{firstName}}` 与 `{{data.firstName}}` 等价；
    嵌套路径 `data.profile.tier` 可用，但 dashboard 不太好发现。
    类型由「项目内首次出现的非 null 值」推断：number / boolean / date（`YYYY-MM-DD` 或完整 ISO 8601）/ string（兜底）；
    推断结果一经确定对该项目粘滞，类型混乱时最稳的重置方式是 `DELETE /contacts/fields/:field` 后重新导入。
  source_page: https://docs.useplunk.com/guides/custom-fields
  aliases: [custom fields, contact data, data object]
  related: [type inference, reserved keys, "persistent: false", memberOfSegment]

- term: reserved keys（保留键）
  kind: plunk-specific
  definition: |
    两层含义。(1) Plunk 在联系人级别管理、可读但**不能通过 `data` 设置**的键：`email`、`createdAt`、`updatedAt`、
    `subscribed`、`snoozedUntil`（要改请用请求里的专用字段或 PATCH）。
    (2) 会被从任何 `data` 载荷里**静默过滤掉**的键：发送了既不存储、也不报错
    —— contacts 概念页列为 `id`、`plunk_id`、`plunk_email`、`unsubscribeUrl`、`subscribeUrl`、`manageUrl`；
    custom-fields 与 `/v1/send` 文档在这份清单里额外包含 `email`；CSV 导入的「保留列名」又是另一套
    （`id`、`subscribed`、`createdAt`、`updatedAt` + 三个 URL 变量）。口径不完全一致，见 glossary-notes。
  source_page: https://docs.useplunk.com/concepts/contacts
  aliases: [保留键, reserved keys, silently filtered keys]
  related: [special keys, locale, unsubscribeUrl, subscribeUrl, manageUrl]

- term: special keys（locale）
  kind: plunk-specific
  definition: |
    `locale` 是「软保留键」：像普通自定义字段一样存于 `data` 之下，但 Plunk 会读取它用于本地化。
    取值 ISO 639 / BCP 47（如 `en`、`fr`、`zh-CN`），用于覆盖项目默认语言（营销邮件退订页脚、托管 unsubscribe /
    subscribe / preferences 页面）。联系人 locale 不匹配任一受支持语言时回落到项目默认，两者都没设则用 English。
    模板里可用 `{{locale}}` 分支自己的文案——Plunk **不会**自动翻译你自己的 subject / body。
  source_page: https://docs.useplunk.com/concepts/contacts
  aliases: [locale field]
  related: [localization, reserved keys]

- term: "persistent: false（非持久化值）"
  kind: plunk-specific
  definition: |
    把值写成 `{ value, persistent: false }`，该值只参与本次渲染，不落库到联系人。
    典型用途：密码重置码、magic link、一次性 token。
    在 `/v1/track`、`POST /contacts`、`PATCH /contacts/:id`、`/v1/send` 的 `data` 里都可用。
    文档提示：不要把密钥放进 `data`（dashboard 与有 API 权限的人都能看到），一次性凭据应用非持久化值。
  source_page: https://docs.useplunk.com/concepts/contacts
  aliases: [non-persistent values, one-shot values]
  related: [data.* 自定义字段, null, empty string]

- term: 特殊值处理（null / "" / 原语）
  kind: plunk-specific
  definition: |
    创建或更新联系人时对值的特殊约定：空字符串 `""` **被忽略**（既不存也不覆盖已有值）；
    `null` **删除**该字段；其它值正常存储/更新。这套规则同样适用于 `/v1/send` 与 `/v1/track` 的 `data`。
  source_page: https://docs.useplunk.com/concepts/contacts
  aliases: [special value handling]
  related: ["persistent: false", data.* 自定义字段]

- term: subscribed（订阅状态）
  kind: plunk-specific
  definition: |
    联系人上的布尔值，决定 TA 会收到哪些邮件：MARKETING / HEADLESS 类型的模板、campaign、automation 会跳过
    `subscribed: false` 的人；TRANSACTIONAL 一律送达（即使走 `/v1/send` 用 marketing 模板也到不了已退订者）。
    默认值按入口不同而不同：`/v1/track` 新建联系人默认 subscribed(true)；`POST /contacts` 默认 false；
    `/v1/send` 新联系人默认 false；CSV 导入的联系人默认 subscribed（会让从未 opt-in 的人进列表）。
    更新联系人时**省略 `subscribed` 表示保持原状态**，不等价于传 false。
    每次翻转都会自动打事件：true → `contact.subscribed`，false → `contact.unsubscribed`。
  source_page: https://docs.useplunk.com/concepts/contacts
  aliases: [订阅状态, subscription state, opt-in / opt-out]
  related: [MARKETING, TRANSACTIONAL, HEADLESS, snooze, snoozedUntil]

- term: snooze（暂停订阅）
  kind: plunk-specific
  definition: |
    退订之外的「次级选项」：收件人在托管 unsubscribe / preferences 页选择暂停 2 周 / 1 月 / 6 月 / 1 年
    （API 取值 `2_weeks`、`1_month`、`6_months`、`1_year`），窗口期内收不到任何邮件，到期由后台任务自动重新订阅。
    它是**真实的 opt-out，而非更软的 opt-out**：整个窗口 `subscribed: false`，campaign / workflow / 经 `/v1/send`
    发出的 marketing 模板都按该标志过滤，不存在「压着以后补发」。
    事件复用既有名字并带 `reason`：暂停时 `contact.unsubscribed` + `{reason:"snooze", duration, snoozedUntil}`；
    到期回来时 `contact.subscribed` + `{reason:"snooze_expired"}`。偏好页的 "resume" 链接或访问 `{{subscribeUrl}}` 可提前结束。
    任一对 subscribed 的其它变更（正式退订、重新订阅、硬退订/投诉抑制、PATCH、CSV 导入、批量操作、
    UPDATE_CONTACT 步骤）都会**取消进行中的 snooze**，且被抑制的地址不会被 snooze 复活。
    用途是压低投诉率：想要「少收」而不是「不收」的人不必去点垃圾邮件按钮。
    Snooze 计入 campaign 的 unsubscribe 数（不算 "unsubscribedCount" 之外的隐藏）。 dashboard 里带 Snoozed 徽章。
    自建退订界面可用同一个免鉴权入口：`POST /contacts/public/:id/snooze?e=<emailId>`，body `{ duration: "6_months" }`。
  source_page: https://docs.useplunk.com/guides/unsubscribe-pages
  aliases: [暂停, snoozing]
  related: [snoozedUntil, subscribed, manageUrl, "`?e=` attribution"]

- term: snoozedUntil
  kind: plunk-specific
  definition: |
    联系人的保留字段，记录 snooze 何时结束；未 snooze 时为 `null`。
    它是区分「snooze」与「永久退订」的唯一标志，用途是排定回归时间——它不参与是否发信的判定
    （发信只看 `subscribed: false`）。
  source_page: https://docs.useplunk.com/concepts/contacts
  aliases: [snooze end timestamp]
  related: [snooze, subscribed, reserved keys]

## Segment 与过滤器 DSL

- term: segment（DYNAMIC / STATIC）
  kind: plunk-specific
  definition: |
    具名联系人分组，可用于 campaign 定向与 workflow 触发。两种 type（默认 `DYNAMIC`）：
    DYNAMIC 由 `condition` 布尔树决定成员（必填 `condition`），后台会定期重算，可选（开启 `trackMembership` 时）
    发出 entry/exit 事件，成员不可通过 API 增删（`POST /segments/:id/members` 会 400）；
    STATIC 手工维护成员列表（`condition` 被忽略 / 更新时静默忽略），只能通过 API 增删，且 API 增删**不会**触发 entry/exit。
    API 创建时 `type` 省略即 DYNAMIC。
  source_page: https://docs.useplunk.com/concepts/segments
  aliases: [audience segment, 分群]
  related: [condition, trackMembership, audienceType, memberCount]

- term: condition / filter DSL（过滤条件树）
  kind: plunk-specific
  definition: |
    同一套 JSON 结构在三处复用：**动态 segment 的 `condition`**、**campaign `audienceType: FILTERED` 的
    `audienceCondition`**、**workflow 的 `CONDITION` 步骤**——「学一次，到处能用」。
    结构：顶层 `{ logic: "AND"|"OR", groups: [] }`；每个 group 内含 `filters: [{field, operator, value?, unit?}]`，
    组内条件之间按 AND 组合，顶层 logic 再把 groups 连起来；group 上再挂 `conditions`（同形状）即可嵌套，
    表达 "A AND (B OR C)" 而无需压平逻辑。
  source_page: https://docs.useplunk.com/guides/segment-filters
  aliases: [filter tree, audience condition, audienceCondition, filter expression]
  related: [logic, groups, filters, filter fields, operators, unit]

- term: logic / groups / filters（三件套）
  kind: plunk-specific
  definition: |
    `logic`：顶层与各嵌套层的连接词，`AND` 或 `OR`。
    `groups`：条件组数组，组内多条 filter 之间固定为 AND，组与组之间由该层 `logic` 连接。
    `filters`：具体条件对象，形如 `{ field, operator, value, unit? }`；需要 `value` 的操作符
    （equals/notEquals/contains/notContains/greaterThan/lessThan/greaterThanOrEqual/lessThanOrEqual/within/olderThan/
    triggeredWithin/triggeredOlderThan/notTriggeredWithin）；需要 `unit`（`days`/`hours`/`minutes`）的操作符
    （within/olderThan/triggeredWithin/triggeredOlderThan/notTriggeredWithin）；
    exists/notExists/triggered/notTriggered/memberOfSegment/notMemberOfSegment 什么都不额外要。
  source_page: https://docs.useplunk.com/guides/segment-filters
  aliases: [filter shape]
  related: [condition / filter DSL, filter fields, unit]

- term: filter fields（过滤字段命名空间）
  kind: plunk-specific
  definition: |
    `field` 的值按前缀区分查询对象：`email`（联系方式1330匹配时对 email 大小写不敏感，对 data.* 精确）、
    `subscribed`、`createdAt` / `updatedAt`、`data.<path>`（支持嵌套）、`event.<eventName>`（自定义事件）、
    `email.<activity>`（sent / delivered / opened / clicked / bounced / complained）、
    `segment.<segmentId>`（另一个 segment 的成员身份，配 `memberOfSegment` / `notMemberOfSegment`）。
  source_page: https://docs.useplunk.com/guides/segment-filters
  aliases: [namespace prefix]
  related: [condition / filter DSL, memberOfSegment, exists / notExists]

- term: operators（过滤操作符）
  kind: plunk-specific
  definition: |
    按字段类型分组。文本：equals / notEquals / contains / notContains；
    是/否（boolean）：equals / notEquals；日期：equals / notEquals / greaterThan / lessThan /
    greaterThanOrEqual / lessThanOrEqual / within / olderThan（值为纯日期 `YYYY-MM-DD` 时按整个 UTC 日比较）；
    数字：equals / notEquals / greaterThan / lessThan / greaterThanOrEqual / lessThanOrEqual（不需 unit）；
    存在性：exists / notExists；事件与邮件行为：triggered / notTriggered / triggeredWithin /
    triggeredOlderThan / notTriggeredWithin；segment 成员：memberOfSegment / notMemberOfSegment。
  source_page: https://docs.useplunk.com/guides/segment-filters
  aliases: [filter operators]
  related: [unit, filter fields]

- term: unit
  kind: plunk-specific
  definition: |
    相对时间操作符（`within`、`olderThan`、`triggeredWithin`、`triggeredOlderThan`、`notTriggeredWithin`）
    必须搭配的时间单位，取值 `"days"` / `"hours"` / `"minutes"`。
    自定义日期字段要用 `within` / `olderThan`，值必须以完整 ISO 8601 字符串（如 `2026-05-06T12:00:00Z`）存储，
    Unix 时间戳或其它格式无法正确比较。
  source_page: https://docs.useplunk.com/guides/segment-filters
  aliases: [time unit]
  related: [operators, ISO 8601]

- term: trackMembership
  kind: plunk-specific
  definition: |
    动态 segment 上的布尔开关（"Track membership changes"）。开启后联系人进出该 segment 时 Plunk 会发出
    `segment.<slug>.entry` / `segment.<slug>.exit` 事件，可用于驱动 workflow。
    成员面向 examine：创建或修改 filter 时成员数立即更新，之后由后台任务按固定节奏重算（diff 新旧集合 → 记录 entry/exit → 发事件），
    因此 dashboard 的 `memberCount` 可能滞后几分钟。立刻要数或要立即触发事件请用两个 endpoint：
    `POST /segments/:id/refresh`（只刷新计数，便宜，不发事件）与 `POST /segments/:id/compute`（完整重算并发 pending 的 entry/exit 事件）。
    建议在被别的 segment 引用的 segment 上开启，让成员直接查表而不是每次重算。
  source_page: https://docs.useplunk.com/concepts/segments
  aliases: [Track membership changes, membership tracking]
  related: [segment slug entry/exit 事件, memberCount]

- term: segment slug entry/exit 事件
  kind: plunk-specific
  definition: |
    开启 `trackMembership` 后发出的事件：`segment.<slug>.entry`（加入）/ `segment.<slug>.exit`（离开）。
    `<slug>` 由 segment 名称派生：小写化、去变音符号与标点、空白替换为连字符、连续连字符合并——
    `"VIP Customers"` → `segment.vip-customers.entry`。**改名会改变事件名**，因此要选能产出稳定 slug 的名字。
    可作 workflow 触发器；webhook 默认载荷中这两个事件带 `{ segmentId, segmentName }`。
    注意 `segment.` 前缀另有他用：过滤字段里的 `segment.<segmentId>` 表示「引用了另一个 segment」。三者别混。
  source_page: https://docs.useplunk.com/concepts/segments
  aliases: [segment entry event, segment exit event]
  related: [trackMembership, reserved event names, internal events]

- term: memberCount
  kind: plunk-specific
  definition: |
    Segment 上缓存的成员数，由后台任务刷新而非每次请求计算，因此可能比真实状态滞后几分钟；
    把它当近似值，需要精确值时调用 `POST /segments/:id/refresh`。
  source_page: https://docs.useplunk.com/api-reference/segments/createSegment
  aliases: [cached member count]
  related: [trackMembership]

## 模板与模板语言

- term: template（邮件模板）
  kind: plunk-specific
  definition: |
    存在 Plunk dashboard 里、可被 campaign、workflow（SEND_EMAIL 步骤）与 `/v1/send` 复用的邮件定义：
    `name`(必填, 内部名)、`description`、`subject`(必填, 支持模板变量)、`body`(必填, HTML, 支持变量)、
    `from`(必填, 必须在已验证域名上，否则 403)、`fromName`、`replyTo`、`type`(必填, 默认 MARKETING)、
    `style`(编辑器元数据, API 创建时可忽略)。
    用于 `/v1/send` 时，请求可以覆盖 `subject` / `body` / `from` / `fromName` / `replyTo`——模板值充当默认值。
    还有 `POST /templates/:id/duplicate`（复制）与 `GET /templates/:id/usage`（查被哪些 campaign、workflow 步骤引用）。
  source_page: https://docs.useplunk.com/concepts/templates
  aliases: [email template, tpl_]
  related: [MARKETING, TRANSACTIONAL, HEADLESS, Liquid, always available variables]

- term: MARKETING（模板类型）
  kind: plunk-specific
  definition: |
    尊重 opt-out（跳过已退订联系人），且 Plunk **自动追加**本地化的退订页脚（含 `{{unsubscribeUrl}}`），
    无需自己做任何事；你自己再放一个 `{{unsubscribeUrl}}` 也可以覆盖/并存的链接。
    三种 type 在投递优先级上没有差别，不要为了可投递性去挑类型。
  source_page: https://docs.useplunk.com/concepts/templates
  aliases: [marketing template]
  related: [TRANSACTIONAL, HEADLESS, unsubscribeUrl, "`?e=` attribution"]

- term: TRANSACTIONAL（模板类型）
  kind: plunk-specific
  definition: |
    不尊重 opt-out（发给任何联系人，无视订阅状态），也不自动追加任何退订入口。
    这是唯一能触达已退订联系人的类型——即便走 `/v1/send` 端点，只要用 marketing 模板就到不了退订者，想必达请换 transactional。
     inbound 自动回复推荐用它（避免对方以后退订就回不了）；double opt-in 的确认邮件也必须用 transactional。
  source_page: https://docs.useplunk.com/concepts/templates
  aliases: [transactional template]
  related: [HEADLESS, MARKETING, transactional email（/v1/send）]

- term: HEADLESS（模板类型）
  kind: plunk-specific
  definition: |
    尊重退订（和 marketing 一样会跳过已退订联系人），但 Plunk 不追加退订页脚；
    必须在邮件正文里自己提供退订入口（用 `{{unsubscribeUrl}}` 或 `{{manageUrl}}`）。
    适合想完全自控退订 UI / 品牌呈现的场景（文档建议：把自家产品内的退订控件链到同一个 `{{unsubscribeUrl}}`）。
  source_page: https://docs.useplunk.com/concepts/templates
  aliases: [headless template]
  related: [MARKETING, TRANSACTIONAL, unsubscribeUrl, manageUrl]

- term: always available variables（常驻模板变量）
  kind: plunk-specific
  definition: |
    无论联系人有没有自定义数据都可用的变量：`{{id}}`（联系人唯一 ID）、`{{email}}`、
    `{{unsubscribeUrl}}`、`{{subscribeUrl}}`、`{{manageUrl}}`；另外 `{{subscribed}}`（订阅状态 true/false）
    与 `{{locale}}`（ISO 639 语言码）也可直接用。未设置的变量渲染为空字符串，既不报错也不会把占位符漏进邮件。
  source_page: https://docs.useplunk.com/concepts/templates
  aliases: [template variables, "{{variable}}"]
  related: [unsubscribeUrl / subscribeUrl / manageUrl, Liquid, "`??` fallback"]

- term: Liquid
  kind: industry-standard
  definition: |
    Shopify / Zendesk / Netlify 使用的模板语言（liquidjs）。Plunk 的 subject 与 body 都用它渲染，
    因此除 `{{变量}}` 外还支持条件（if/elsif/else、unless、case/when）、循环（for，含 forloop.index/.first/.last/.length，
    支持 limit/offset/reversed）与过滤器（`upcase`、`capitalize`、`default`、`date`、`plus/minus/times/divided_by`、
    `join`、`truncate` 等完整 LiquidJS 列表）。
    Plunk 侧的特殊行为：① 采用 **JavaScript 真假判断**而非 Liquid 更严格的规则（空字符串为假，`{% if firstName %}` 对 `""` 为 false）；
    ② 渲染**故意宽容**，模板永远不会导致发送失败——未知变量渲染为空、未知过滤器跳过原值输出、存储的模板解析失败时回退到
    纯 `{{variable}}` 替换；③ 保存模板/campaign 时 Liquid 解析错误会返回 400 并给出行列，但 `/v1/send` 的
    inline `subject`/`body` **不做这层校验**（best-effort 渲染）；④ 拒绝 `{% include %}`、`{% render %}`、`{% layout %}`；
    ⑤ 直接输出数组 `{{items}}` 会把每项包进 `<li>`（为兼容旧行为保留，建议显式写 `{% for %}`）；
    ⑥ 一次 arrays 也支持 `assign` / `capture` 计算值。
  source_page: https://docs.useplunk.com/guides/template-language
  aliases: [LiquidJS, template language]
  related: ["`??` fallback", raw tag, always available variables]

- term: "`??` fallback"
  kind: plunk-specific
  definition: |
    Plunk 专有简写：变量缺失或为 null 时给一个字面量兜底，形如 `{{firstName ?? 'there'}}`。
    Liquid 的 `default` 过滤器是更通用的形式，还能回退到另一个变量：`{{firstName | default: nickname}}`。
    两者都把「缺失 / null / false / 空值」视为未设置。
    注意两页示例口径略不一致：templates 页写带引号 `{{firstName ?? 'there'}}`，template-language 页写不带引号
    `{{firstName ?? there}}` 并说明「带引号的兜底值不再泄漏引号，这是本指南一贯描述的行为」。
  source_page: https://docs.useplunk.com/guides/template-language
  aliases: [fallback values, default filter]
  related: [Liquid, always available variables]

- term: raw tag
  kind: industry-standard
  definition: |
    `{% raw %}...{% endraw %}`，让 `{{ }}` / `{% %}` 原样出现在邮件里。
    Plunk 侧为何必须记住：引入 Liquid 后，**正文中成对出现的标签会被真正执行**——原本会原样打印的
    `Write {% if x %}…{% endif %} to branch` 现在会被求值成空白，想要字面展示就得裹 `{% raw %}`。
    不成对的写法（`Use {% if x %} in docs`）仍解析失败、照旧存活。
  source_page: https://docs.useplunk.com/guides/template-language
  aliases: ["{% raw %}", escaping template syntax]
  related: [Liquid]

## Campaign

- term: campaign（邮件活动）
  kind: plunk-specific
  definition: |
    一次性的批量群发（newsletter、公告、发布、促销），在某个时间点发一次（立即或定时）然后冻结；
    与 workflow 的区别就是「只发一次」。字段：`name`、`description`、`subject`、`body`、`from`（须已验证域名，否则 403）、
    `fromName`、`replyTo`、`type`（MARKETING/TRANSACTIONAL/HEADLESS，语义同模板类型，默认 MARKETING）、
    `audienceType`（ALL/SEGMENT/FILTERED）、`segmentId`（SEGMENT 时必填）、`audienceCondition`（FILTERED 时必填，
    结构与 segment 的 `condition` 相同）、`scheduledFor`（ISO 8601，为 null 表示触发 send 时立即发）。
    发送后用 `POST /campaigns/:id/send`（后台排队发送）、`POST /campaigns/:id/cancel`（只能停未交接给对方邮件服务器的部分）、
    `POST /campaigns/:id/test`（单地址试发，body 是 `{ email: "..." }`，注意是单个地址不是数组）、
    `POST /campaigns/:id/duplicate`（复制成新的 DRAFT）。发送后保留反范式化统计字段。
  source_page: https://docs.useplunk.com/concepts/campaigns
  aliases: [email campaign, broadcast, camp_]
  related: [campaign status, audienceType, campaign stats]

- term: campaign status（DRAFT / SCHEDULED / SENDING / SENT / CANCELLED）
  kind: plunk-specific
  definition: |
    生命周期：DRAFT（已创建未调度/未发送，可编辑）→ SCHEDULED（有 `scheduledFor`，可取消）→
    SENDING（队列正在处理收件人，可取消）→ SENT（完成，统计最终）；
    CANCELLED 是可从 DRAFT / SCHEDULED / SENDING 进入的终态。
    SENT 或 CANCELLED 之后不能再编辑或重发，只能 duplicate 出可编辑副本。
  source_page: https://docs.useplunk.com/concepts/campaigns
  aliases: [campaign lifecycle]
  related: [campaign（邮件活动）]

- term: audienceType（ALL / SEGMENT / FILTERED）
  kind: plunk-specific
  definition: |
    Campaign 受众类型，决定哪个配套字段必填：
    `ALL`——项目内所有已订阅联系人，两个字段都不要；`SEGMENT`——当前属于某个 segment 的所有成员，必须给 `segmentId`；
    `FILTERED`——内联过滤条件（与 segment 过滤器相同 JSON 形状），必须给 `audienceCondition`，适合一次性受众不想存成可复用 segment。
    MARKETING / HEADLESS campaign 自动跳过已退订联系人；TRANSACTIONAL campaign 则会投递给所有匹配受众、不管订阅状态。
    注意别把它和同请求里的 `type`（邮件内容类型 MARKETING/TRANSACTIONAL/HEADLESS）混淆，API 文档专门标注了这一点。
  source_page: https://docs.useplunk.com/concepts/campaigns
  aliases: [audience condition]
  related: [condition / filter DSL, campaign（邮件活动）]

- term: campaign stats（统计口径）
  kind: plunk-specific
  definition: |
    `GET /campaigns/:id/stats` 返回 dashboard 上的计数与比率（recipients / sent / delivered / opened / clicked /
    bounced / complained / unsubscribed + 对应比率）。所有比率的分母都是 `sentCount`；计数每次调用从底层邮件与事件重算，
    是权威值而非最终一致。
    三个「流失」列各自对同一收件人只计一次：`bouncedCount`（由 SES bounce webhook 写 `Email.bouncedAt`，硬退订会抑制联系人）、
    `complainedCount`（`Email.complainedAt`，会抑制联系人）、`unsubscribedCount`（由退订链接上的 `?e=` 归因）。
    bounce/complaint 虽然也会把联系人翻成退订，但**只进各自的列，绝不进 `unsubscribedCount`**；
    dashboard 开关、CSV 导入、`PATCH /contacts/:id`、以及该参数存在之前发出的邮件里的链接导致的退订无法归因，也就不计入任何 campaign。
    投诉率要看总数而非百分比（服务商按千分之几计量，持续高于约 0.1% 会危及发信），dashboard 因此在这张卡上先显示计数。
  source_page: https://docs.useplunk.com/concepts/campaigns
  aliases: [unsubscribedCount, complainedCount, bouncedCount]
  related: [credits, bounce, complaint, "`?e=` attribution"]

## Workflow

- term: workflow（自动化工作流）
  kind: plunk-specific
  definition: |
    图结构的自动化：由一份 workflow 记录 + step 图 + step 之间的 transition + 每个联系人一份 execution 组成；
    每个进入的联系人独立跑一遍。单个 trigger 决定联系人如何进入；一旦 structural 结构在跑就不让改（见 locking）。
    API 结构上映射为 `/workflows`、`/workflows/:id/steps`、`/workflows/:id/transitions`、`/workflows/:id/executions` 四组端点。
    通过 API 创建时，trigger 默认是给的 `EVENT`，且**一律 `enabled: false`**——不会有 execution 启动。
  source_page: https://docs.useplunk.com/concepts/workflows
  aliases: [automation, journey, wf_]
  related: [triggerType, step types, transition, execution, allowReentry, enabled]

- term: triggerType（EVENT / MANUAL / SCHEDULE）
  kind: plunk-specific
  definition: |
    EVENT——匹配事件被 track 到时触发（自定义事件、系统事件如 `email.received` / `contact.subscribed`、segment entry/exit）；
    MANUAL——只有在显式调用 `POST /workflows/:id/executions` 时才启动；
    SCHEDULE——由 `triggerConfig`（如 cron 表达式）定义的周期调度。
    API 创建时默认为 EVENT 加你给的事件名；要用 MANUAL / SCHEDULE 得创建后用 `PATCH /workflows/:id` 改 `triggerType` + `triggerConfig`。
    **触发事件名在第一次 execution 之后不可更改**（记得用 `waitlist.joined` 这类命名空间化的名字，别用以后想复用的泛名），
    但其它触发配置（延迟、步骤内条件）仍可编辑。
  source_page: https://docs.useplunk.com/concepts/workflows
  aliases: [trigger, triggerConfig]
  related: ["STEP: TRIGGER", internal events]

- term: step types（TRIGGER / SEND_EMAIL / DELAY / WAIT_FOR_EVENT / CONDITION / WEBHOOK / UPDATE_CONTACT / EXIT）
  kind: plunk-specific
  definition: |
    八种步骤。TRIGGER：自动创建的入口，持有触发配置（`eventName`）；
    SEND_EMAIL：用模板给联系人发邮件（需 `templateId`，可选 `from` 覆盖），变量取自联系人 data + execution context；
    DELAY：固定时长暂停（需 `amount` 与 `unit`：minutes/hours/days）；
    WAIT_FOR_EVENT：等到指定事件被 track 到，带超时兜底（需 `eventName`、`timeout` 秒）；
    CONDITION：按联系人或事件数据分支，有两条 tagged `yes` / `no` 的出边（表达式同 segment filters 形状）；
    WEBHOOK：向外部 HTTPS 端点发 JSON（contact + execution context），`url`、header 值、`body` 都支持 `{{变量}}`；
    UPDATE_CONTACT：patch 联系人 data（如打标签 `{ stage: "activated" }`）；
    EXIT：终止 execution，可选记录 `reason` 作为 `exitReason`。
    注意 `POST /workflows/:id/steps` 的路由表里只列了后七种——TRIGGER 是系统自动建的。
  source_page: https://docs.useplunk.com/concepts/workflows
  aliases: [workflow steps]
  related: [transition, branch, WEBHOOK step, /workflows/fields]

- term: /workflows/fields
  kind: plunk-specific
  definition: |
    `GET /workflows/fields` 返回可用于 `CONDITION` 步骤过滤器的字段集合（联系人字段 + 事件字段）。
  source_page: https://docs.useplunk.com/api-reference/overview
  aliases: [workflow condition fields]
  related: ["STEP: CONDITION", condition / filter DSL]

- term: transition
  kind: plunk-specific
  definition: |
    Workflow 图的边。大多数步骤的 transition 就是一个简单的「下一个」指针；`CONDITION` 步骤的 transition 额外带 `branch`
    判别器（`yes` / `no`），引擎据此决定走哪条路。删除链条中间的步骤时，必须在
    `DELETE /workflows/:id/steps/:stepId` 上带 `?splice=true` 才会自动重连两侧，否则图会断开。
  source_page: https://docs.useplunk.com/concepts/workflows
  aliases: [edges]
  related: [branch, splice]

- term: branch
  kind: plunk-specific
  definition: |
    `CONDITION` 步骤出边上的判别标记，`"yes"` 或 `"no"`；新建 transition 时对 CONDITION 步骤必须带上。
  source_page: https://docs.useplunk.com/api-reference/overview
  aliases: [yes/no branch]
  related: ["STEP: CONDITION", transition]

- term: splice
  kind: plunk-specific
  definition: |
    `DELETE /workflows/:id/steps/:stepId?splice=true` 的查询参数，删除步骤时自动重连它前后的 transition；
    不带则留下一个断开的图。
  source_page: https://docs.useplunk.com/api-reference/overview
  aliases: ["?splice=true"]
  related: [transition]

- term: execution（WorkflowExecution 状态）
  kind: plunk-specific
  definition: |
    每个联系人进入 workflow 都会创建一份 execution，状态有：RUNNING（正在或即将处理某一步）、
    WAITING（卡在 DELAY 或 WAIT_FOR_EVENT）、COMPLETED（成功跑完图）、EXITED（撞到 EXIT 步骤，`exitReason` 记录原因）、
    FAILED（不可恢复错误，如 webhook 重试后仍非 2xx、模板找不到）、CANCELLED（经 API 或 dashboard 手动取消）。
    每份 execution 带一个 `context` JSON 对象，渲染模板与求值条件时会与联系人的 `data` 合并；
    用 `POST /workflows/:id/executions` 手动启动时可传初始 `context`，这就是在不把变量落库到联系人的前提下
    做「每 execution 参数化」（优惠券码、推荐人）的方式。
  source_page: https://docs.useplunk.com/concepts/workflows
  aliases: [WorkflowExecution, RUNNING, WAITING, COMPLETED, EXITED, FAILED, CANCELLED]
  related: [context, allowReentry, locking active workflows]

- term: context（execution context）
  kind: plunk-specific
  definition: |
    Execution 上携带的 JSON 对象，在渲染模板、求值 CONDITION 条件时与联系人 `data` 合并。
    手动启动 execution 时传入，用来做「不落库到联系人」的每次执行变量（优惠券码、推荐人名字）；
    这些 key 在模板与 WEBHOOK 步骤里也能以 `{{key}}` 直接引用。
  source_page: https://docs.useplunk.com/concepts/workflows
  aliases: [execution context]
  related: [execution, WEBHOOK step, "persistent: false"]

- term: allowReentry
  kind: plunk-specific
  definition: |
    Workflow 上的开关（默认 `false`），控制同一联系人能否多次进入同一 workflow。
    关：适合「只触一次」的序列（欢迎邮件、onboarding）；开：适合可重复参与的旅程（反复提醒、重复催促）。
  source_page: https://docs.useplunk.com/concepts/workflows
  aliases: [re-entry]
  related: [enabled, execution]

- term: enabled
  kind: plunk-specific
  definition: |
    Workflow 的开关；创建时一律为 `false`，**翻转成 `true` 之前不会有任何 execution 启动**——这是
    「我的 workflow 怎么不触发」最常见的原因。可用 `PATCH /workflows/:id` 修改。
  source_page: https://docs.useplunk.com/concepts/workflows
  aliases: [enable toggle]
  related: [allowReentry, locking active workflows]

- term: locking active workflows（编辑锁定）
  kind: plunk-specific
  definition: |
    只要 workflow 还有 RUNNING / WAITING 的 execution，它就被锁住不能做结构性改动（增删步骤、改触发事件、换触发类型）。
    要做破坏性变更必须先二选一：禁用 workflow 并等活动 execution 跑完，或 `POST /workflows/:id/executions/cancel-all` 取消所有在飞的。
    外观性改动（改名、编辑步骤引用的模板内容）不受此限。删除 workflow 同理要求 active execution 先取消或完成。
  source_page: https://docs.useplunk.com/concepts/workflows
  aliases: [edit lock, structural edits]
  related: [enabled, execution]

- term: WEBHOOK step
  kind: plunk-specific
  definition: |
    Workflow 中的一种步骤，也是 Plunk 发出站 webhook 的唯一方式（先在 dashboard 建 workflow → 选触发事件 → 加 Webhook 步骤）。
    可变 interpolation 作用域：`url`、header 值、`body` 都支持 `{{变量}}`，可用变身 Myc 同 SEND_EMAIL 模板的集合，
    外加 webhook 独有的 `event` 命名空间（`{{event.<key>}}` 取触发事件载荷字段）与 execution `context` 里的 key。
    HTTP method **不可模板化**，必须是字面动词；url 必须含静态 scheme（`http://` / `https://`），占位符不能替换 scheme。
    交付特性：默认 POST + JSON，超时 **10 秒**（长活端点应「接住→入队→立刻返回 2xx」）；
    最多跟随 5 次重定向，且**每一跳**都重新校验 SSRF 规则；必须公网可达（loopback、RFC 1918 等私有地址不投递）；
    只接受 http/https；**没有自动重试**（非 2xx 或超时即失败该步骤，重试语义要用 WAIT_FOR_EVENT / 备用分支自己搭，
    同时自己实现幂等）；真实性校验用共享密钥 header（优于 IP 白名单，因为出口 IP 会变）。
    body 留空时发送默认载荷 `{ contact, workflow, execution, event }`。
  source_page: https://docs.useplunk.com/guides/webhooks
  aliases: [webhook step, outbound webhook]
  related: [default webhook payload, context, "`{{event.<key>}}`"]

- term: default webhook payload
  kind: plunk-specific
  definition: |
    未配置自定义 body 时发送的 JSON：`{ contact: {email, subscribed, data}, workflow: {id, name},
    execution: {id, startedAt}, event: {...} }`。
    所有 email 类事件共享基础字段：`subject`、`from`、`fromName`、`messageId`（provider 侧 ID）、
    `emailId`（Plunk 邮件记录 ID，与 `POST /v1/send` 返回的一致，用于把 webhook 与你的发送请求对应起来）、
    `templateId`、`campaignId`、`sourceType`（`TRANSACTIONAL` / `CAMPAIGN` / `WORKFLOW` / `INBOUND`），
    再叠加各事件专属字段（sentAt / deliveredAt / openedAt+opens+isFirstOpen / link+clickedAt+clicks+isFirstClick /
    bounceType+bouncedAt 或 transientBounce / complainedAt / 入站那套）。
    `contact.subscribed` 与 `contact.unsubscribed` 默认 `event` 为空对象 `{}`；但若退订由 bounce / complaint 自动触发，
    会带上 `{ reason: "bounce" | "complaint" }`。自定义事件则原样带上你 track 时传的 data。
  source_page: https://docs.useplunk.com/guides/webhooks
  aliases: [webhook payload]
  related: [emailId, internal events, WEBHOOK step]

## 发送通道、事件与 API

- term: transactional email（/v1/send）
  kind: plunk-specific
  definition: |
    直接通过 API 的一对一邮件（密码重置、订单确认、通知）。`POST /v1/send`：
    内容要么给 `template` ID，要么同时给 `subject` + `body`（二选一）；transy winds `from` 除非模板已配。
    `to` 可为 string / `{name,email}` / 数组；**数组时每个收件人顺序处理、各自 upsert 联系人并各自渲染，没有批量语义**；
    发送永远立即，要定时请用 Campaign。
    附件：默认上限 10 个 / 10MB 合计，整条消息 ≤ 40MB，base64 传入 `attachments[]`（`filename` / `content` / `contentType` /
    可选 `contentId` + `disposition: inline` 用于正文嵌入 `<img src="cid:...">`）。
    message 格式：body 填 HTML，Plunk 自动生成并同时发送纯文本版本（没有让你设纯文本的字段）。
    `/v1/send` 的 inline subject/body **不做 Liquid 语法预校验**，解析不了的部分回退成纯 `{{variable}}` 替换而不报错。
    `subscribed` 字段：新联系人默认 false；已存在的联系人省略则保持原状态，显式传 true/false 会改变并发对应事件。
  source_page: https://docs.useplunk.com/api-reference/public-api/sendEmail
  aliases: [send email API, transactional send, /v1/send]
  related: [emailId, Idempotency-Key, TRANSACTIONAL, "persistent: false"]

- term: emailId
  kind: plunk-specific
  definition: |
    Plunk 的邮件记录 ID，`POST /v1/send` 的 200 响应里 `data.emails[].email` 返回它，形如 UUID。
    所有 email 事件的 webhook 载荷里都带同一个 `emailId`，用它把 webhook 事件与你的发送请求精确对应起来，
    无需靠「邮箱 + 时间戳」匹配，也不用为了拿 provider `messageId` 而额外监听 `email.sent`。
    同一个值也作为退订链接上的 `?e=` 参数，用于把退订归因到具体邮件。
  source_page: https://docs.useplunk.com/api-reference/public-api/sendEmail
  aliases: [Plunk email record ID]
  related: ["`?e=` attribution", default webhook payload]

- term: /v1/track
  kind: plunk-specific
  definition: |
    给联系人打事件（找不到就先建联系人再记事件），可用 pk_ 或 sk_ 调用。事件可作 workflow 触发、segment 过滤、受众过滤。
    `subscribed` 语义：新联系人默认 subscribed(true)，已有联系人保持原状除非显式传值——
    要给已退订者打事件又不想把他重新订阅，就显式传 `false`。重复 track 同一事件会产生新记录，靠 Idempotency-Key 才能拒绝重复。
  source_page: https://docs.useplunk.com/api-reference/public-api/trackEvent
  aliases: [track event, /events/track]
  related: [reserved event names, internal events, Idempotency-Key]

- term: reserved event names
  kind: plunk-specific
  definition: |
    不可手动 track 的事件名模式，命中会返回 `VALIDATION_ERROR` 且错误码 `reserved_event`：
    任何匹配 `email.*`、`contact.subscribed`、`contact.unsubscribed`、`segment.<slug>.entry`、`segment.<slug>.exit` 的名字。
    这些由 Plunk 自己发出。
  source_page: https://docs.useplunk.com/api-reference/public-api/trackEvent
  aliases: [internal-only events]
  related: [internal events, segment slug entry/exit 事件]

- term: internal events（系统事件）
  kind: plunk-specific
  definition: |
    Plunk 自动生成、可作 workflow 触发或 hub 的邮件 lemudicionary 事件：
    邮件类 `email.sent`、`email.delivery`、`email.open`（首次打开）、`email.click`（首次点击）、`email.bounce`、
    `email.complaint`、`email.received`（需入站配置）；联系人类 `contact.subscribed` / `contact.unsubscribed`；
    segment 类 `segment.<name>.entry` / `segment.<name>.exit`（名字取 slug 版本）。
    注意 `email.open` / `email.click` 是「首次」语义，而 webhook 载荷同时给出总次数 `opens` / `clicks` 与 `isFirstOpen` / `isFirstClick`。
  source_page: https://docs.useplunk.com/guides/webhooks
  aliases: [system events]
  related: [reserved event names, open tracking, click tracking]

- term: pk_ / sk_（public key / secret key）
  kind: plunk-specific
  definition: |
    每个项目恰好两把 API key，走 `Authorization: Bearer` 的同一套格式。
    `pk_`（public）**只能调 `POST /v1/track`**，可以放浏览器 / 移动端；`sk_`（secret）能调其它所有端点，只能服务端用。
    前缀与端点不匹配（如拿 pk_ 调 `/v1/send`）返回 401 + `INVALID_API_KEY`。
    项目归属由 key 自动推导，请求里不需要额外的 project ID 参数。
    两把钥匙是成对存在的：轮换（`POST /users/@me/projects/:id/regenerate-keys`）**同时重生成两把**，
    无法只轮换一把，且旧 key 立即失效、没有宽限期。
  source_page: https://docs.useplunk.com/guides/api-keys
  aliases: [public key, secret key, API keys]
  related: [Idempotency-Key, MCP tools]

- term: Idempotency-Key
  kind: industry-standard
  definition: |
    请求头，保证操作「至多执行一次」；新 key 正常处理，本项目已用过的 key 会以 409 +
    `IDEMPOTENCY_KEY_REUSED` **拒绝**（注意：是拒绝，不是重放原响应——拿不到原来的邮件/事件 ID，需要就在首次成功时自己记下来）。
    `details` 里给出 `originalRequest` / `originalRequestAt` / `originalStatusCode`，后者为 `null` 表示原请求还在飞、这次抢输了。
    Plunk 侧的特殊行为：① 只在 `POST /v1/track` 与 `POST /v1/send` 两个公开写端点上支持；
    ② key 是任意 1–255 个可打印 ASCII 字符，**范围是整个项目而非某个端点**（在 track 用过的 key 不能拿去 send），
    但不同项目可用同一个字符串；③ 有效期 24 小时，过期才可复用（自托管用 `IDEMPOTENCY_KEY_TTL_HOURS` 调）；
    ④ 失败是否释放：2xx 保留、4xx 释放（校验/权限错误在写库前就被拒，可修好重试同一 key）、5xx 保留（可能已完成一半，这正是要它保护的场景）；
    ⑤ 多收件人时 key 覆盖整次请求而非每个收件人——半途 5xx 时部分人已收到，key 仍被占用以防盲目重试重复发送；想要逐个控制就一人一请求一 key。
  source_page: https://docs.useplunk.com/guides/idempotency
  aliases: [idempotency, at-most-once]
  related: [data retention 窗口, pk_ / sk_]

- term: "error envelope（`{success, error{...}, timestamp}`）"
  kind: plunk-specific
  definition: |
    统一错误响应：`{ success: false, error: { code, message, statusCode, requestId, errors[], details, suggestion }, timestamp }`。
    成功响应用 `{ success: true, data: {...} }`，但只有 public API（`/v1/send`、`/v1/track`、`/v1/verify`）才是这层包裹——
    dashboard 端点（contacts / templates / campaigns / segments / workflows 等）直接返回资源本身、无 wrapper。
    `requestId` 同时出现在响应头 `X-Request-ID`（成功响应也有），用于排障与联系支持。
    机器可读 code 包括 `INVALID_API_KEY`、`MISSING_AUTH`、`VALIDATION_ERROR`（含字段级 `errors`）、
    `RESOURCE_NOT_FOUND` 系列、`CONFLICT`、`RATE_LIMIT_EXCEEDED`、`BILLING_LIMIT_EXCEEDED`、`UPGRADE_REQUIRED`、
    `IDEMPOTENCY_KEY_REUSED` 等。建议先判 `success` 再按 `code` 分支（而不是只看 HTTP 状态码）。
  source_page: https://docs.useplunk.com/api-reference/errors
  aliases: [error format, requestId, X-Request-ID]
  related: [pagination, rate limit bucket]

- term: pagination（cursor vs offset）
  kind: industry-standard
  definition: |
    大多数列表端点用 **cursor** 分页：`limit`（默认 20，最大 100）+ `cursor`，响应返回 `{ data, cursor, hasMore, total }`；
    `total` 只在首页（未传 cursor 时）给出，后续页为 0。
    少数端点用 **offset** 分页：`GET /campaigns` 与 `GET /templates` 用 `page` / `pageSize`，响应带 `total` / `totalPages`；
    此外 `GET /segments/:id/contacts` 也是基于页码（`page`、`pageSize` 上限 100）。
    `GET /segments` 则完全不分页，直接返回整个数组。
  source_page: https://docs.useplunk.com/api-reference/overview
  aliases: [cursor pagination, offset pagination]
  related: [error envelope]

- term: rate limit bucket（限流桶）
  kind: industry-standard
  definition: |
    Plunk 侧有两套口径，需并列看待、不可混用：
    ① API overview 说「API requests 1000 requests/minute per project」，超限返回 429 `RATE_LIMIT_EXCEEDED`。
    ② 自托管环境变量文档说限流是存在 Redis 里的**按项目** token bucket（不是按 IP，所以一个客户跑飞不会吃掉别人的预算），
    **默认关闭**（`RATE_LIMIT_ENABLED=false`，升级也不会突然开始拒绝既有流量），开启后三个桶各自有持续速率与突发上限：
    `/v1/track` 100/s + burst 200、`/v1/send` 20/s + burst 50（另受 billing limit 约束）、contacts 写操作 50/s + burst 100
    （POST/PATCH/DELETE 共享一个预算，因为同步集成本来就是从同一条上游变更流驱动增删改）；单个 bucket 设为 0 可单独关掉。
    每个响应带 `RateLimit-Limit` / `RateLimit-Remaining` / `RateLimit-Reset`，被限流的请求返回 429 + `Retry-After`；
    Redis 不可达时放行而非拒绝。
  source_page: https://docs.useplunk.com/self-hosting/environment-variables
  aliases: [RATE_LIMIT_ENABLED, token bucket, RateLimit-* headers]
  related: [billing limits, error envelope]

- term: /v1/verify
  kind: industry-standard
  definition: |
    邮箱校验端点（`POST /v1/verify`，需 sk_）：检查格式、MX 记录、一次性域名、别名/转发服务、plus addressing、
    个人邮箱服务商、域名是否存在（NS）、是否 typo 并给出建议；返回 `valid` / `isDisposable` / `isAlias` / `isTypo` /
    `isPlusAddressed` / `isPersonalEmail` / `domainExists` / `hasWebsite` / `hasMxRecords` / `suggestedEmail` / `reasons[]`。
    Plunk 侧用法：在注册时用它防止坏地址进列表、也是控制退订率的手段之一；自托管里 `VERIFY_EMAIL_ON_SIGNUP=true`
    会用它做同样的检查。
  source_page: https://docs.useplunk.com/api-reference/public-api/verifyEmail
  aliases: [email verification]
  related: [bounce, /v1/track]

## 退订、归因与本地化

- term: unsubscribeUrl / subscribeUrl / manageUrl
  kind: plunk-specific
  definition: |
    三个**每次发送时自动生成**的 per-recipient 签名 URL 变量，在任意可退订邮件（marketing / headless 模板）里可用：
    `{{unsubscribeUrl}}` → 一键退订页（停止接收营销邮件）；`{{subscribeUrl}}` → 重新订阅页（之前退订过的人 opt back in）；
    `{{manageUrl}}` → 偏好设置页（查看当前状态并可双向切换）。
    是否自动出现在邮件里取决于模板类型：MARKETING 自动追加本地化页脚（含 unsubscribeUrl）；HEADLESS 不追加，
    你必须自己放 `{{unsubscribeUrl}}` 或 `{{manageUrl}}`，否则收件人无法退订；TRANSACTIONAL 不追加也不该需要。
    走 `/v1/send` 时同样按所引用模板的类型决定（无模板 inline 发送 → 无自动页脚）。
    这三个名字同时也属于会被从 `data` 静默过滤掉的保留键（发送时会重新生成）。
  source_page: https://docs.useplunk.com/guides/unsubscribe-pages
  aliases: [subscription management URLs, List-Unsubscribe]
  related: [reserved keys, snooze, "`?e=` attribution", MARKETING, HEADLESS]

- term: "`?e=` attribution（退订归因）"
  kind: plunk-specific
  definition: |
    Plunk 在发送时生成的链接上带的 `?e=<emailId>` 参数，标识这封邮件；出现在 `List-Unsubscribe` 头、
    自动注入的 marketing 页脚、以及 `{{unsubscribeUrl}}` / `{{subscribeUrl}}` / `{{manageUrl}}` 三个变量上。
    收件人动作时把这封邮件记录到事件上，活动流里就能指出这次退订来自哪个 campaign / workflow；
    参数会跨托管页面传递，所以「先落到偏好页再退订」仍能归因到起始邮件。
    故意不可归因的情形：链接在 email 行存在之前就渲染出来的（campaign body 的 `{{unsubscribeUrl}}` 是在队列时而非发送时渲染）、
    背后没有邮件的变化（dashboard 开关、CSV 导入、`PATCH /contacts/:id`）、以及该参数存在之前发出的邮件。
    自建退订面时可自己拼 `?e=`：它是**建议性的**，Plunk 会校验该邮件属于该联系人，真正决定变更的是路径里的 contact id，
    缺失或错误只损失归因。
  source_page: https://docs.useplunk.com/guides/unsubscribe-pages
  aliases: [attribution parameter, e parameter]
  related: [unsubscribeUrl / subscribeUrl / manageUrl, campaign stats, emailId]

- term: localization（支持语言与回落）
  kind: industry-standard
  definition: |
    Plunk 只翻译**它自己控制的字符串**：自动注入的 marketing 退订页脚、托管的 unsubscribe / subscribe / preferences 页面；
    你自己的 subject 与 body 不自动翻译（自己做「每种语言一个模板」或「一个模板里用 `{{locale}}` 分支」两种模式）。
    回落链：联系人 `locale` → 项目默认语言 → English。
    支持 14 种：en、nl、fr、de、es、it、pt、pl、cs、bg、hi、zh-CN、zh-TW、zh-HK。
  source_page: https://docs.useplunk.com/guides/localization
  aliases: [i18n, supported languages]
  related: [special keys（locale）, unsubscribeUrl / subscribeUrl / manageUrl]

## 收件（Inbound）

- term: inbound email（email.received）
  kind: plunk-specific
  definition: |
    Plunk 可接收发往已验证域名任意地址的邮件并存进项目。流程：解析并存储为入站 Email 记录 →
    把发件人建成（或更新为）联系人、**默认 subscribed** → 在该联系人上打 `email.received` 事件，供任意 workflow 监听。
    HTML 正文存储前会被消毒（剥离 script、iframe、事件处理器、`javascript:` URI），无 HTML 时保留纯文本原文。
    限制：catch-all only（靠 `event.to` 在工作流里分流）、不保留附件、不保留原始 MIME、不做 threading、
    单封 > 40MB 直接拒收；同一域名在多个项目里都验证时，每封入站邮件会投递给**每个**项目（各自独立记账/upsert/事件）。
    如果你的系统中枢不是 Plunk，收完再 Discount 的事件，别忘了入口自动订阅这点可能不是你想要的
    （用 `UPDATE_CONTACT` 步骤设 `subscribed: false`）；自动回复要防 loop（丢弃 `event.from` 属于自己域名的）。
  source_page: https://docs.useplunk.com/guides/receiving-emails
  aliases: [receiving emails, email.received]
  related: [verdict, MX record, credits, billing limits]

- term: verdict（spam / virus / spf / dkim / dmarc）
  kind: industry-standard
  definition: |
    入站邮件上的安全与认证判定，`email.received` 事件载荷里给出：`spamVerdict`、`virusVerdict`
    （取值 `PASS` / `FAIL` / `GRAY` / `PROCESSING_FAILED`）、`spfVerdict`、`dkimVerdict`、`dmarcVerdict`。
    Plunk 侧的关键行为：**事件永远会发出，Plunk 不基于这些判定阻断邮件**，过滤要自己在 workflow 的第一个
    CONDITION 步骤里做（敏感收件箱建议直接丢弃 spam/virus 为 FAIL 的）；判定是**建议性的**，不强制执行。
    另外要注意 body 属于不可信输入，即便已消毒仍可能有钓鱼链接与社会工程内容。
  source_page: https://docs.useplunk.com/guides/receiving-emails
  aliases: [authentication verdicts]
  related: [inbound email, SPF, DKIM, DMARC]

- term: "`{{event.<key>}}`"
  kind: plunk-specific
  definition: |
    WEBHOOK 步骤独有的变量命名空间，暴露触发事件载荷的字段（如 `{{event.subject}}`、`{{event.from}}`、`{{event.body}}`、
    `{{event.messageId}}`、`{{event.timestamp}}`、以及入站判定的 `{{event.spamVerdict}}` 等）。
    在 inbound 自动回复里也可以把 To 写成 `{{event.from}}`、subject 写成 `Re: {{event.subject}}`。
  source_page: https://docs.useplunk.com/guides/webhooks
  aliases: [event namespace]
  related: [WEBHOOK step, inbound email, verdict]

## 域名验证与可投递性

- term: DKIM
  kind: industry-standard
  definition: |
    DomainKeys Identified Mail：给邮件加数字签名以证明未被篡改，防冒充。
    Plunk 侧：域名验证要求添加 **3 条 CNAME**（由 dashboard 给出具体值）。与 SPF 配合构成完整认证，主流服务商基本强制。
  source_page: https://docs.useplunk.com/guides/verifying-domains
  aliases: [DomainKeys Identified Mail]
  related: [SPF, DMARC, MAIL FROM domain, verdict]

- term: SPF
  kind: industry-standard
  definition: |
    Sender Policy Framework：声明哪些邮件服务器被授权代表你的域名发信。
    Plunk 侧：验证需要 **1 条 TXT** 记录。铁律：**一个域名只能有一条 SPF TXT**——如果已经在用
    Google Workspace / Microsoft 365 / 其它发信平台，必须把 Plunk 的 include **合并**进已有记录，
    不能新增第二条（两条会同时失效）。例：`v=spf1 include:_spf.google.com include:<Plunk 的 include> ~all`。
  source_page: https://docs.useplunk.com/guides/verifying-domains
  aliases: [Sender Policy Framework]
  related: [DKIM, DMARC, MAIL FROM domain, verdict]

- term: DMARC
  kind: industry-standard
  definition: |
    Domain-based Message Authentication, Reporting and Conformance：告诉收件方 SPF/DKIM 失败时怎么办、报告发往哪里。
    Plunk 不强制要求，但 Gmail / Yahoo / Microsoft 现在对 bulk sender 普遍期望要有；
    加 `_dmarc.yourdomain.com` 的 TXT，如 `v=DMARC1; p=none; rua=mailto:dmarc-reports@yourdomain.com`，
    先 `p=none` 观察两周确认所有合法发信都通过了，再收紧到 `p=quarantine`、`p=reject`。
  source_page: https://docs.useplunk.com/guides/verifying-domains
  aliases: []
  related: [SPF, DKIM, MAIL FROM domain, verdict]

- term: MAIL FROM domain
  kind: industry-standard
  definition: |
    自定义退信信封域名（通常用子域如 `mail.yourdomain.com`），让 bounce envelope 地址与发信域名对齐，
    改善 DMARC 对齐，部分收件箱要求这样才能完全通过。启用后需在该子域上再加 MX + TXT 各一条（dashboard 给出确切值）。
    自托管侧由 `MAIL_FROM_SUBDOMAIN` 控制构造出的主机名（默认前缀 `plunk`，即 `plunk.yourdomain.com`）；
    若该子域已被 R2/CDN 自定义域名占用，得改前缀，因为 MAIL FROM 主机名需要的 MX + TXT 无法与 CNAME 共存。
  source_page: https://docs.useplunk.com/guides/verifying-domains
  aliases: [envelope sender domain]
  related: [DMARC, MX record, SES IAM policy]

- term: MX record（bounce MX vs inbound MX）
  kind: industry-standard
  definition: |
    MX 记录，但 Plunk 里有两条用途完全不同的 MX，必须区分：
    **bounce MX**（域名验证四件套之一）——用于接收 Plunk 发出邮件的投递反馈（退信、投诉），
    从而自动标记退订/投诉联系人、维护发信声誉，是**必做**的；
    **inbound MX**（可选，启用收信功能时才加）——用于**接收**发往该域名的邮件。两者可以只配其一，也可以都配。
    坑：一个域名只能有一个主收信 MX；若 apex 域已经在用 Google Workspace / Microsoft 365 收信，
    把 MX 指向 Plunk 会让现有邮箱挂掉——通常的解法是让 Plunk 收信走子域（如 `mail.` / `support.`），
    且该子域也要先在 Plunk 完成发信验证其 MX 才会被接受。入站 MX 只在「发信完全验证」的域名上工作。
  source_page: https://docs.useplunk.com/guides/verifying-domains
  aliases: [bounce handling MX, inbound MX]
  related: [inbound email, bounce, complaint, MAIL FROM domain]

- term: domain verification（发信域名验证）
  kind: plunk-specific
  definition: |
    通过 Plunk 发信的前置要求：在项目设置的 Domains 页添加域名 → 加上 DNS 记录（DKIM 3×CNAME + SPF 1×TXT +
    bounce MX 1×MX，建议 DMARC，可选自定义 MAIL FROM）→ Plunk 后台定期校验
    （通常几分钟，最长 72 小时随 DNS 传播；也可 `GET /domains/:id/verify` 强制立即检查，否则每 5 分钟一次）。
    模板与 campaign 的 `from`、以及 `/v1/send` 的发件域名都必须位于本项目已验证且 DNS 校验完成的域名上，否则 **403**。
    注意：域名的增删只能通过 dashboard（MCP 也刻意不暴露），自托管时 Plunk 会自动向 AWS SES 处理验证与 DKIM。
  source_page: https://docs.useplunk.com/guides/verifying-domains
  aliases: [verified domain]
  related: [DKIM, SPF, MX record, template（邮件模板）]

- term: bounce（Permanent / Transient / Undetermined）
  kind: industry-standard
  definition: |
    邮件无法投递。Plunk 侧：每次 bounce 都在联系人上记 `email.bounce` 事件，**只有 Permanent（硬退）会自动退订联系人**，
    Transient（软退，如邮箱满 / 服务器不可用 / greylist）不改订阅状态，Undetermined（上游无法归类）也不改；
    两者都可以在 workflow 或 webhook 里靠载荷的 `bounceType` 字段区分来分支。
    只有 Permanent 计入项目退信率并触发自动退订；Transient 仅作可见性记录（webhook 里表现为 `transientBounce: true`、
    且不写 `bouncedAt`）。
    自托管还有 `AUTO_PROJECT_DISABLE`（默认 true）在退/投诉率超阈值时自动暂停项目。
  source_page: https://docs.useplunk.com/guides/list-hygiene
  aliases: [hard bounce, soft bounce, email.bounce]
  related: [complaint, MX record, SES simulator]

- term: complaint
  kind: industry-standard
  definition: |
    收件人在邮箱服务商里把邮件标记为垃圾邮件。Plunk 侧：收到投诉后该联系人**一律自动退订**并打 `email.complaint` 事件。
    投诉比退信严重得多——服务商按千分之几计量，持续高于约 0.1% 会危及发信。
    自动退订时 `contact.unsubscribed` 事件会带 `{ reason: "bounce" | "complaint" }`，
    所以sync-unsubscribes 类 webhook 可以用 `event.reason ?? 'user_action'` 区分来源。
  source_page: https://docs.useplunk.com/guides/list-hygiene
  aliases: [spam complaint, email.complaint]
  related: [bounce, default webhook payload, snooze]

- term: plain text alternative
  kind: industry-standard
  definition: |
    每封 Plunk 发出的邮件都同时带你所写 HTML 与由 HTML 自动生成的纯文本版本，不必自己写或传
    （transactional、campaign、test send 都这样；`/v1/send` 没有让你设纯文本的字段）。
    这是为了躲 HTML-only 的垃圾信号（SpamAssassin 的 `MIME_HTML_ONLY`、`MPART_ALT_DIFF`）：
    生成的文本保留链接的文字与目的地（`View invoice [https://...]`）、标题/段落/列表各占一行、展平布局表格、
    丢弃图片与追踪像素，要求与 HTML 说同样的话（否则会触发它本想规避的那批过滤器）。
    如果邮件完全没有可读文本（比如只有一张图），Plunk 宁可省略文本部分也不附带一个空的。
  source_page: https://docs.useplunk.com/guides/list-hygiene
  aliases: [multipart alternative]
  related: [bounce, open tracking]

- term: SES simulator
  kind: industry-standard
  definition: |
    亚马逊 SES 邮箱模拟器地址（`@simulator.amazonses.com`），用来安全演练 bounce / complaint 处理：
    `bounce@`（硬退）、`complaint@`（投诉）、`success@`（成功投递）、`ooto@`（休假自动回复）、
    `suppressionlist@`（如同在抑制名单上的硬退）。
    这些发送在 Plunk 里完全可见（事件会发、workflow 与 webhook 会跑、联系人动态里会显示），
    但**不计入**你的退信率与投诉率，因此永远不会触发警告或暂停；AWS 侧也把它们排除在自己的指标之外。
  source_page: https://docs.useplunk.com/guides/list-hygiene
  aliases: [mailbox simulator]
  related: [bounce, complaint]

- term: open tracking / click tracking（与 tracking 模式）
  kind: industry-standard
  definition: |
    open tracking = 在邮件底部插一个不可见像素，客户端加载即记为 `email.open`；
    click tracking = 重写正文链接走跳转，点击时记 `email.click` 再转发到目的地。
    局限：均依赖客户端行为——禁 remote image 就没 open，某些客户端/企业安全网关会预取产生假 open，
    Apple Mail Privacy Protection 会掩盖真实 open，某些安全工具预扫链接产生假 click；重写也让邮件源码里的 URL 与目标不一致。
    Plunk 侧：**tracking 是「全有或全无」**，没有开关只追踪打开或只追踪点击；在 Settings → Tracking 三选一
    ——Enabled（所有邮件）、Disabled（任何邮件都不追）、Marketing only（只追营销类：campaign 以及使用 marketing
    或 headless 模板的 workflow 步骤；事务类——`/v1/send` 走 transactional 模板、或 `TRANSACTIONAL` campaign——不追踪）。
    自托管时 Marketing only **只有在配置了 no-tracking configuration set 时可选**（见 configuration set）。
  source_page: https://docs.useplunk.com/guides/tracking
  aliases: [tracking modes, tracking pixel, link rewriting]
  related: [internal events, configuration set（SES）]

## 计费与数据保留

- term: credits（计费口径）
  kind: plunk-specific
  definition: |
    按个数、按前的统一使用量单位。事务 `/v1/send` 1 credit/收件人、campaign 1 credit/收件人、
    workflow 的 SEND_EMAIL 步骤 1 credit/次、入站邮件 1 credit/封；
    **带一个或多个附件的邮件是 2 credits**（不按附件大小加价，100KB 还是 10MB 都一样）。
    免费层每月 1,000 封（含入站计入同一额度，且营销邮件带 Plunk 品牌页脚）；升级后 $0.001/封、无月费无承诺。
  source_page: https://docs.useplunk.com/concepts/billing
  aliases: [usage-based pricing, what counts as an email]
  related: [billing limits, 402 BILLING_LIMIT_EXCEEDED]

- term: billing limits（分类上限）
  kind: plunk-specific
  definition: |
    在 Settings → Billing → Limits 对四个类目**各自独立**设月度上限：Transactional（`/v1/send`）、
    Campaigns、Workflows（SEND_EMAIL 步骤）、Inbound。可以只设一个、也可以都不设。
    目的是防止单一来源跑飞（比如保证就算 campaign 超额 HA er误ければ，事务能力仍然可用）。
    超限行为：出站 API 调用返回 **402** + `BILLING_LIMIT_EXCEEDED`（应用应优雅处理：排队或提示用户）；
    **入站邮件则对该项目静默丢弃**（不排队、不重放），直到限额重置（每个计费周期开始）。
    接近或触达时有邮件/站内通知。额度 Api：`GET/PUT /users/@me/projects/:id/billing-limits`。
  source_page: https://docs.useplunk.com/concepts/billing
  aliases: [per-category caps, spending caps]
  related: [credits, 402 BILLING_LIMIT_EXCEEDED, inbound email]

- term: 402 BILLING_LIMIT_EXCEEDED
  kind: plunk-specific
  definition: |
    超出分类上限或套餐总额时出站调用返回的状态码与错误码；另一个 402 是 `UPGRADE_REQUIRED`（功能需要升级套餐）。
    与之相对，入站超限不是错误而是静默丢弃。
  source_page: https://docs.useplunk.com/api-reference/errors
  aliases: [PAYMENT_REQUIRED]
  related: [billing limits, credits]

- term: data retention 窗口
  kind: plunk-specific
  definition: |
    Plunk 永久保留「发过什么」的记录，只是把占体积的部分按计划老化：
    邮件元数据与指标：无限期；**渲染后的邮件 HTML：90 天**（到期清空 body 但邮件行仍在，邮件-door 与目标отариус，
    只是 dashboard 里不能再预览内容，且清空不可逆——要永久归档请在发送时自己留存，或从模板 + 联系人数据重建）；
    **API 请求日志：30 天**（只影响请求级排障）；**幂等键：24 小时**（过期才 reusable，这是幂等机制的一部分而非存储考虑）。
    自托管：清理是 worker 里的定时任务（body 每天 04:00、请求日志每天 03:00、幂等键每小时整点，时区由容器 `TZ` 决定），
    从不跑 worker 的部署会无界增长；90/30 天窗口是编进代码的（不可配），只有幂等 TTL 可用 `IDEMPOTENCY_KEY_TTL_HOURS` 调。
    首次升级后运行会一次性清掉全部 90 天以前的 body，大库会很慢，且 Postgres 不会自动收缩表（需 `VACUUM FULL`）。
  source_page: https://docs.useplunk.com/guides/data-retention
  aliases: [retention policy, cleanup jobs]
  related: [Idempotency-Key, emailId]

## 自托管与 AI 集成

- term: configuration set（tracking / no-tracking）
  kind: industry-standard
  definition: |
    AWS SES 的 configuration set。自托管需要**两个**：
    `plunk-tracking`（事件目的地包含 Sends, Deliveries, Opens, Clicks, Bounces, Complaints）用于开启打开/点击追踪；
    `plunk-no-tracking`（只含 Sends, Deliveries, Bounces, Complaints）用于不带追踪地发送。
    环境变量：`SES_CONFIGURATION_SET` 与 `SES_CONFIGURATION_SET_NO_TRACKING`。
    Plunk 侧的关键耦合：**不配 no-tracking 的话，项目里的 tracking 开关会被隐藏**——也就是说自托管实例上
    "Marketing only" 模式只有配了它才可用。SES 侧的事件统一经 SNS topic 推到 `https://api.yourdomain.com/webhooks/sns`。
  source_page: https://docs.useplunk.com/self-hosting/email-setup
  aliases: [SES_CONFIGURATION_SET, SES_CONFIGURATION_SET_NO_TRACKING]
  related: [open tracking / click tracking, SES sandbox, inbound email]

- term: SES sandbox
  kind: industry-standard
  definition: |
    新 AWS SES 账号默认处于 sandbox：只能发往**已验证**地址、200 封/24 小时、1 封/秒。
    只适合测试，生产要在 SES console（Account dashboard → Request production access）申请出沙，
    AWS 通常在短问卷后 24 小时内批准。
  source_page: https://docs.useplunk.com/self-hosting/email-setup
  aliases: [sandbox mode, production access]
  related: [configuration set（SES）, rate limit bucket]

- term: MCP tools
  kind: industry-standard
  definition: |
    官方 MCP server（npm 包 `@plunk/mcp`），让 AI agent 在 Plunk 项目里干活，hosted 与自托管都支持，需要 `sk_`。
    环境变量：`PLUNK_API_KEY`、`PLUNK_API_URL`（自托管时指向自己的 API 域）、`PLUNK_PUBLIC_KEY`（有则让打事件只需一次请求，
    没有则先用 sk_ 建/更新联系人再记事件，两种途径联系人都会建）、`PLUNK_READ_ONLY`、`PLUNK_ALLOW_UNCONFIRMED_SENDS`、
    `PLUNK_MCP_API_KEY` / `PLUNK_MCP_API_URL`（同名冲突时的替代，注意自托管里 Plunk 服务端自己也用 `PLUNK_API_KEY`
    发平台通知邮件，两者混在同一环境时必须用后者区分）。
    工具集：只读类（`plunk_list_contacts`、`plunk_get_contact`、`plunk_verify_email`、`plunk_list_templates`、
    `plunk_list_campaigns`、`plunk_get_campaign`、`plunk_get_campaign_stats`、`plunk_list_segments`、`plunk_list_domains`、
    `plunk_check_domain`）与写入类（create/update/subscribe/unsubscribe/delete contact、`plunk_send_email`、`plunk_track_event`、
    `plunk_create_template`、`plunk_create_campaign`、`plunk_test_campaign`、`plunk_send_campaign`、`plunk_cancel_campaign`、
    `plunk_create_segment`）。
    安全阀：发送活动或给多于一个收件人发信时**必须由人确认**（无法自己跳过的 prompt，客户端不能提示就不能发送）、
    `PLUNK_READ_ONLY=true` 时写工具根本不注册、以及 billing / 删项目 / 轮换密钥这些需要 dashboard 会话的操作**刻意不提供**
    （域名的增删同理——那两个端点跳过 admin 角色校验，暴露会让 agent 获得非 admin 成员没有的权限）。
    另外 `plunk_test_campaign` 的收件人必须是该项目成员，其它地址会被拒。
  source_page: https://docs.useplunk.com/guides/mcp-server
  aliases: ["@plunk/mcp", PLUNK_READ_ONLY, PLUNK_ALLOW_UNCONFIRMED_SENDS]
  related: [pk_ / sk_, domain verification, campaign（邮件活动）]

- term: worker / api / web（SERVICE）
  kind: plunk-specific
  definition: |
    自托管容器可通过 `SERVICE` 环境变量只跑某一部分：`api`（仅 API）、`worker`（仅 worker，跑队列任务、定时清理、
    segment 重算等）、`web`（仅 dashboard）、`all`（默认，全部）。
    健康检查端点 `GET /health` 返回 `{status, time, uptime}`，用于编排器的存活/就绪探针。
  source_page: https://docs.useplunk.com/self-hosting/docker
  aliases: [SERVICE env, /health]
  related: [data retention 窗口, rate limit bucket]

- term: phishing detection（钓鱼检测）
  kind: plunk-specific
  definition: |
    可选的 AI 钓鱼检测，需要 OpenRouter API key（`OPENROUTER_API_KEY`，模型默认 `anthropic/claude-3-haiku`）。
    按 `PHISHING_DETECTION_SAMPLE_RATE`（默认 0.1，即 10%）抽样送 LLM 分析；命中任一条件即自动禁用项目：
    单封置信度 ≥ `PHISHING_CONFIDENCE_THRESHOLD`（默认 95），或在 `PHISHING_CUMULATIVE_WINDOW_MS`（默认 1 小时）内
    累积被标记 ≥ `PHISHING_CUMULATIVE_THRESHOLD`（默认 3）封。检测历史存 Redis，跨 worker 实例共享。
  source_page: https://docs.useplunk.com/self-hosting/environment-variables
  aliases: [OPENROUTER_API_KEY]
  related: [AUTO_PROJECT_DISABLE, bounce, complaint]

- term: AUTO_PROJECT_DISABLE
  kind: plunk-specific
  definition: |
    自托管环境变量（默认 `true`）：项目的退信率或投诉率超过阈值时自动暂停该项目；设为 `false` 则改为手动管理项目状态。
    项目被禁用时 API 返回 403 `PROJECT_DISABLED`，且该项目不再处理入站邮件。
  source_page: https://docs.useplunk.com/self-hosting/environment-variables
  aliases: [project suspension]
  related: [bounce, complaint, phishing detection]

## 端到端事实（容易被忽略的常识）

- term: CSV import（CSV 导入规则）
  kind: plunk-specific
  definition: |
    `POST /contacts/import`（multipart，≤5MB，UTF-8），必须有表头且第一列是 `email`，其余列映射为 `data.<列名>`；
    返回 `jobId`，轮询 `GET /contacts/import/:jobId` 直到 `completed` / `failed`，响应给 `totalRows` / `imported`（新建数）/
    `updated`（被 patch 的已有联系人数）/ `skipped` / `errors[]`（含行号与原因）。
    类型推断细节：`true/false/yes/no`（大小写不敏感）→ boolean；纯整数小数 → number；
    **前导零（`01234`）、带 `+` 号、科学计数法保持字符串**，以免 ID / 邮编 / 电话被破坏。
    已存在的邮箱按行 **update 合并 data**，不产生重复也不整体覆盖——所以重跑同一份 CSV 是幂等的、
    修好某列重传即可批量更正。注意导入的联系人**默认 subscribed**（与 `POST /contacts` 默认 false 不同）。
  source_page: https://docs.useplunk.com/guides/importing-contacts
  aliases: [bulk import, import job]
  related: [contact（联系人）, data.* 自定义字段, type inference]

- term: type inference
  kind: plunk-specific
  definition: |
    自定义字段的类型（string / number / boolean / date）按「项目内第一次见到的非 null 值」推断，之后对该项目粘滞：
    即便某个 key 被推断成 number，后续仍可存字符串值，但**推断类型不会变**（只影响 segment 过滤 UI 显示什么控件）。
    想在 `data.<key>` 上用 `within` / `olderThan` 或 `{{ signupDate | date: ... }}`，值必须以完整 ISO 8601 字符串存储。
    混了类型时最稳的重置方式：`DELETE /contacts/fields/:field` 删字段后用想要的类型重新导入
    （删除只影响联系人数据，引用它的 segment / workflow 仍存在，但对这个字段的条件将不再匹配任何东西，
    删前先用 `GET /contacts/fields/:field/usage` 查引用）。
    审计用什么字段：`GET /contacts/fields` 返回全部字段、推断类型与覆盖率 `coverage`。
  source_page: https://docs.useplunk.com/guides/custom-fields
  aliases: [inferred type, field type]
  related: [data.* 自定义字段, unit, ISO 8601]

- term: bulk operations（批量操作与 job）
  kind: plunk-specific
  definition: |
    `POST /contacts/bulk-subscribe` / `bulk-unsubscribe` / `bulk-delete`，每次最多 1,000 个**联系人 ID**，
    均为异步队列任务并返回 job ID，用 `GET /contacts/bulk/:jobId` 轮询；每次翻 subscribed 都会走一遍
    `contact.subscribed` / `contact.unsubscribed` 事件，也会清掉进行中的 snooze。
    另有 `POST /contacts/lookup`（一次最多 500 个邮箱做存在性检查）。
  source_page: https://docs.useplunk.com/api-reference/overview
  aliases: [bulk job, jobId]
  related: [contact（联系人）, snooze, subscribed]
