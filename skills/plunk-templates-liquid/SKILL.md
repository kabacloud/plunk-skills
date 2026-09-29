---
name: plunk-templates-liquid
description: |
  当你要新建或修改 Plunk 模板/活动并决定 type（MARKETING/TRANSACTIONAL/HEADLESS），
  或邮件里变量没渲染、渲染出 0/false、条件分支走反、保存报 400（带行列号）、
  从 Shopify/Zendesk 迁移模板、做多语言与附件核算时使用。
  用户提供正文、发信域名与语义（是否尊重退订），产出模板定义、渲染核对清单与本地化选型。
  不负责 /v1/send 端点契约、联系人字段建模、退订状态与活动统计口径。
  triggers: 模板, template, Liquid, 变量没渲染, "??" fallback, truthiness, "{% raw %}", unsubscribe footer, headless, localization, locale, plain text, attachments
metadata:
  cangjie.generated-by: cangjie-tools v2.5.0
  cangjie.capability-id: cap.plunk-docs.templates-and-liquid
  cangjie.capability-revision: 1
  cangjie.bundle-id: bundle.plunk-docs
  cangjie.source-title: Plunk Developer Docs
  cangjie.tags: templates, liquid, rendering, localization
---
# 模板与 Liquid 渲染

## R 原文依据

- [/concepts/templates](https://docs.useplunk.com/concepts/templates)："There are three types of templates in Plunk. Each type is treated at the same priority when sending emails, you should not pick one type over the other based on deliverability or performance."
- [/concepts/templates](https://docs.useplunk.com/concepts/templates)："Marketing: Respects opt-out Yes, footer Yes. Transactional: Respects opt-out No, footer No. Headless: Respects opt-out Yes, footer No — You are responsible for providing an unsubscribe mechanism in the email body. Use `{{unsubscribeUrl}}` or `{{manageUrl}}`."
- [/guides/unsubscribe-pages](https://docs.useplunk.com/guides/unsubscribe-pages)："Marketing: Yes — Plunk appends a localized footer... The footer respects the recipient's `locale`... Sending with no template (just `subject` + `body`) → no auto footer. Sending with a marketing template → footer auto-injected. Sending with a headless or transactional template → no footer."
- [/guides/template-language](https://docs.useplunk.com/guides/template-language)："Custom fields under `data` are available both with and without the prefix... `{{firstName}}` / `{{data.firstName}}`... A variable that isn't set renders as an empty string — it never errors and never leaks the placeholder."
- [/guides/template-language](https://docs.useplunk.com/guides/template-language)："Plunk uses JavaScript truthiness rather than Liquid's stricter rules, because imported contact data very often has a column present but blank. An empty string is falsy, so `{% if firstName %}` is false for a contact whose `firstName` is `\"\"`."
- [/guides/template-language](https://docs.useplunk.com/guides/template-language)："Saving a template or campaign whose syntax Liquid can't parse returns `400` with the line and column of the problem... `/v1/send` is not checked this way... rendered on a best-effort basis."
- [/guides/template-language](https://docs.useplunk.com/guides/template-language)："Rendering itself is deliberately forgiving — nothing about a template can fail a send: An unknown variable renders empty. An unknown filter is skipped... falls back to plain `{{variable}}` substitution."
- [/guides/localization](https://docs.useplunk.com/guides/localization)："If the contact's `locale` doesn't match a supported language, Plunk falls back to the project default. If neither is set, English is used... Your own template subject and body is not translated automatically."

## I 解释

模板是 Plunk 里唯一被 campaign、workflow SEND_EMAIL 步骤与 `/v1/send` 三方共用的内容定义，所以它同时承担三件事：内容（subject/body）、发件人（from/fromName/replyTo）、行为（type 决定合规行为）。

第一层判断是 `type`。它不是"投递优先级"开关——文档明确三种 type 在发送优先级上完全相同，选型的唯一正当依据是语义与合规：**是否该尊重退订**和**要不要 Plunk 自动追加页脚**这两个独立维度。MARKETING 两个都是（页脚还是按收件人 locale 本地化的，即本地化的退订页脚由 Plunk 提供）；TRANSACTIONAL 两个都不（这是唯一能触达已退订联系人的类型）；HEADLESS 尊重退订但不给页脚，因此必须自己在正文放 `{{unsubscribeUrl}}` 或 `{{manageUrl}}`，否则收件人没有退订出口、投诉率上升。同一张矩阵在 campaign 上同样生效（campaign 的 `type` 语义与模板类型一致），因此"用哪个 type"这个决定必须在使用侧做一次，而不是在模板和 campaign 上各做一次。

第二层是"模板类型决定 footer，而不是端点决定 footer"。这一点在 `/v1/send` 上最容易搞错：不带模板只传 subject+body 时没有页脚；引用 marketing 模板时页脚被自动注入；引用 headless 或 transactional 模板时不注入，正文完全由你控制。换言之，事务邮件也会因为引用了 marketing 模板而拿到页脚，反之亦然。

第三层是渲染语义。自定义字段带不带 `data.` 前缀等价，未设置的变量渲染为空串（不报错、也不把占位符漏进邮件）；`??` 只能回退到字面量，Liquid `default` 过滤器还能回退到另一个变量，两者对"未设置"的判定一致——missing / null / false / 空串都算没设置。最关键的非标准行为是 **JS truthiness**：空串为假，所以 `{% if firstName %}` 对 `firstName: ""` 走 else，这与标准 Liquid（空串为真）相反，从 Shopify/Zendesk 迁移模板时分支会走反。

第四层是容错设计：渲染的错误容忍度高于正确性。未知变量渲染为空、未知过滤器跳过原值输出、存储模板在发送时无法解析则整体回退为纯 `{{variable}}` 替换——**任何模板问题都不会导致发送失败**。代价是拼写错误完全静默，只能靠预览发现。与之对应的校验分层是：模板/活动在**保存时**强校验（400 并给出行列号），而 `/v1/send` 的内联 subject/body **完全不校验**（事务正文常由另一个系统生成），坏 Liquid 静默降级为纯替换，请求照样成功。

把这三层摊开，就是排查"变量没渲染出来"的定位表：

| 症状 | 发生在哪一层 | 判据与动作 |
| --- | --- | --- |
| 保存模板/活动报 400，带行列号 | 保存期 | 语法错，照行列号改；这一层能拦住绝大多数低级错误 |
| `/v1/send` 返回成功但条件/循环没生效 | 发送期（内联无校验） | 内联正文不做预检，坏 Liquid 回退为纯替换；改到己侧先渲染或改用保存的模板 |
| 某个变量输出为空、或输出了 `0` / `false` | 渲染期 | 未知变量渲染为空、未知过滤器跳过；falsy 值现在会渲染出来（不再消失） |
| 正文里一段示例文字消失了 | 渲染期（升级行为） | 平衡的标签会被执行，必须用 `{% raw %}` 包裹 |

第五层是本地化与内容形态的边界。Plunk 只翻译**它自己控制的字符串**（营销邮件自动注入的本地化退订页脚、托管的 unsubscribe / subscribe / preferences 页），你自己的 subject 与 body 一律不自动翻译。自建策略有两条：每语言一模板（`welcome-en` / `welcome-fr`，发送前在代码或 workflow 里挑）适合长邮件与规模化；单模板内用 `{{locale}}` 分支适合短文案，长邮件会难维护。locale 走三级回退：联系人 `data.locale`（BCP 47 / ISO 639，须在支持列表内）→ 项目默认语言 → 英语；支持 14 种，三种中文变体（`zh-CN` / `zh-TW` / `zh-HK`）分开，所以必须写完整标签而不是 `zh`。

最后要记住模板是"被引用"的资产，不是孤立的文件：改动前用 `GET /templates/:id/usage` 查它被哪些 campaign 与 workflow 步骤引用，需要变体时用 `POST /templates/:id/duplicate` 复制而不是原地改。字段层面，`name` / `subject` / `body` / `from` 必填，`type` 省略即 MARKETING，`style` 只是 dashboard 拖拽编辑器的元数据、通过 API 创建时可以忽略。

## A1 示例

**示例 1 · 单模板覆盖多语言与多档位**（来自 `/guides/template-language` 的 worked example）

- 输入/问题：要给 3 种语言 × 2 种套餐发同一批续费提醒，按"一个组合一个 campaign"要发 6 次。
- 怎么执行：一个模板里用 `{% if locale == 'es' %}...{% elsif locale == 'fr' %}...{% else %}...{% endif %}` 分支语言，配合 `{{firstName ?? cliente}}` 兜底；再用 `{% case plan %}{% when 'pro' %}...{% when 'free' %}...{% endcase %}` 分支优惠。
- 产出/预期输出：一次发送覆盖全部组合（文档口径："three languages × two offers is one send rather than six"）。渲染的具体文案对照文档未给出完整输出，只给了片段。

**示例 2 · 模板内派生值并截断**（来自 `/guides/template-language`）

- 输入/问题：折扣额度要按 `lifetimeValue` 算出来，且不能超过 25%。
- 怎么执行：`{% assign discount = lifetimeValue | divided_by: 20 %}` → `{% if discount > 25 %}{% assign discount = 25 %}{% endif %}` → `Here's {{discount}}% off`。
- 产出/预期输出：派生值在模板内复用，上限 25%。若 `lifetimeValue` 未设置，按容错规则渲染为空（不会让发送失败）。

**示例 3 · 文档型邮件里的 Liquid 示例被吃掉**（来自 counter-example e42）

- 输入/问题：一封"如何使用模板变量"的说明邮件正文里写了 `Write {% if x %}…{% endif %} to branch`，升级后这段在收件人侧变成空白。
- 怎么执行：把要原样展示的片段包进 `{% raw %}...{% endraw %}`；不成对的写法（如 `Use {% if x %} in docs`）仍然解析失败、会原样保留，无需处理。
- 产出/预期输出：片段按字面显示。若不处理，后果是正文缺块——且**没有任何报错**。

**示例 4 · locale 设了却还是英文**（合成演练，按 `/guides/localization` 的回退链可复算）

- 输入/问题：联系人的 `data.locale` 设成了 `zh`，营销页脚仍是英文。
- 怎么执行：按三级回退链核对——① `zh` 不在 14 种支持语言内（应为 `zh-CN` / `zh-TW` / `zh-HK`）→ 改写成完整 BCP 47 标签；② 若仍不生效，检查 Settings → Project 的项目默认语言是否被改过。
- 产出/预期输出：页脚与托管退订/订阅/偏好页按目标语言渲染。注意自己的 subject/body **不会**被自动翻译，那部分要靠"每语言一模板"或 `{{locale}}` 分支。

**示例 5 · 内联正文的坏 Liquid 静默降级**（来自 counter-example e44）

- 输入/问题：正文由另一个系统拼出后走 `/v1/send` 内联传入，请求返回成功，但收件人看到的内容里条件分支没生效。
- 怎么执行：认清这条链路**没有保存期校验**——无法解析的标记会退化成纯 `{{variable}}` 替换而不是让请求失败。要么在己侧先渲染/校验再发，要么改成引用已保存的模板（保存时会 400 并给出行列号）。
- 产出/预期输出：要么己侧拦截到语法错误，要么模板保存期返回 400。若什么都不做，结果是"请求 200、内容不对、没有任何错误信号"。

**示例 6 · 带附件的事务邮件容量核算**（合成演练，按 `/concepts/transactional-emails` 的阈值可复算）

- 输入/问题：要发一封带 3 个 PDF（合计 8MB）的账单邮件。
- 怎么执行：逐个核对三重上限——数量 ≤10、附件合计 ≤10MB、整封消息 ≤40MB；附件需 base64 编码放进 `attachments` 数组。
- 产出/预期输出：本例三项都通过（3 ≤ 10、8MB ≤ 10MB、整封远低于 40MB）。附件个数或合计体积超限时，文档只说明自托管可用 `MAX_ATTACHMENTS_COUNT` / `MAX_ATTACHMENT_SIZE_MB` 调整前两项，**40MB 是 SES 硬上限、不可调**；超限的精确失败行为文档未说明。

## A2 何时会用

**用户场景**

1. 要新建或修改一个模板/campaign，需要决定 type 是 MARKETING / TRANSACTIONAL / HEADLESS。
2. 邮件里的变量没渲染出来、渲染成了 `0` / `false`、或条件分支走进了相反的分支。
3. 保存模板时报 400（带行列号），或 `/v1/send` 内联正文"没报错但内容不对"。
4. 从 Shopify / Zendesk / 其他 Liquid 实现迁移模板，或 CSV 导入产生了带空格、带 `|` 的字段名。
5. 要做多语言邮件：纠结每语言一个模板还是单模板内 `{{locale}}` 分支，或设了 locale 不生效。
6. 事务邮件需要带附件，或想自己控制 multipart 里的纯文本版本。

**语言信号（中英双写）**

`模板` / `template` / `Liquid` / `liquidjs` / `变量没渲染` / `variable not rendering` / `{{ }}` / `data.` 前缀 / `?? fallback` / `default filter` / `truthiness` / `{% raw %}` / `{% include %}` / `unsubscribe footer` / `headless` / `localization` / `locale` / `plain text` / `attachments`

**与相邻能力的区分**

- 本卡负责：模板内容本身——type 选型、变量与 Liquid 语义、渲染容错与校验分层、本地化策略、附件与纯文本这些"邮件长什么样"的事。
- `plunk-transactional-send` 负责 `/v1/send` 端点契约（收件人、idempotency、emailId、覆盖规则），本卡只说明"模板值作默认值、内联正文不校验"这一层。
- `plunk-campaigns-analytics` 负责 campaign 的受众、生命周期与统计口径；本卡只覆盖 campaign 的 `type`（内容类型）与模板共享的语义。
- `plunk-subscription-consent` 负责退订/订阅状态、托管页与 snooze；本卡只判断"哪个 type 会自动给退订出口"。
- `plunk-contacts-and-fields` 负责 `data.*` 字段怎么建、类型推断与保留键；本卡只负责这些字段在模板里怎么引用。

**按诉求快速分流**：

- "模板保存报 400 / 变量没渲染" → 本卡（渲染与校验层）。
- "发出去的活动数据怎么读" → `plunk-campaigns-analytics`。
- "退订状态怎么改、托管页与 snooze 怎么配" → `plunk-subscription-consent`。
- "字段怎么建、CSV 怎么导" → `plunk-contacts-and-fields`。
- "邮件根本没送到 / 进垃圾箱" → `plunk-domain-deliverability`（认证）与 `plunk-sending-health`（声誉）。

## E 执行步骤

**输入契约**

必需信息：
1. 内容：subject 与 body（HTML），以及要引用的变量名清单。
2. 发件人：`from`（域名必须已在本项目完成验证）、可选 `fromName` / `replyTo`。
3. 语义：这封邮件该不该尊重退订、要不要 Plunk 自动页脚 → 决定 `type`。
4. 目标受众语言（决定是否走本地化分支）。
5. 是否带附件（数量、总体积、编码方式）。

缺失时先问：发信域名是否已验证（未验证必然 403）、有没有必须触达已退订联系人的合规理由（决定能否用 TRANSACTIONAL）、语言集合与文案长度（决定本地化选型）。

**输出契约**

产出：可直接提交的 `POST /templates` 请求体（含显式 `type`）、变量与分支的渲染核对清单、本地化选型结论、附件容量核算结论。验收：① 创建返回 201 且 `type` 与预期一致；② 用指定联系人在预览里核对渲染结果（不是只看保存成功）；③ 三态矩阵与页脚行为逐条对上。

1. **定 type**：先问"是否应尊重退订"与"是否要自动页脚"。尊重退订 + 要页脚 → `MARKETING`；不尊重退订（密码重置、订单确认、double opt-in 确认、入站自动回复）→ `TRANSACTIONAL`；尊重退订但完全自控退订 UI → `HEADLESS`。完成标准：能一句话说清为什么不是另外两个，且**不引用投递率或性能作为理由**。
2. **补退订出口**：若第 1 步选了 HEADLESS，必须在正文显式放 `{{unsubscribeUrl}}` 或 `{{manageUrl}}`（或把自家产品内的退订控件链到同一个 URL）。完成标准：预览里能看到可点击的退订链接；不放即视为缺陷（收件人无路可退）。
3. **写变量与分支**：自定义字段用 `{{name}}` 或 `{{data.name}}` 均可，含空格的字段名用 `{{data["first name"]}}`；需要兜底时用 `{{x ?? '字面值'}}` 或 `{{x | default: 另一个变量}}`。完成标准：列出所有变量并逐项确认"未设置时渲染为空"是否可接受。
4. **按 JS truthiness 复核分支**：凡是 `{% if 字段 %}` 的判断，都要考虑空串为假（CSV 导入常见"有列但为空"）。需要区分"缺失"与"空串"时改用 `??` / `default` 显式处理；需要输出布尔分支时用 `{% unless subscribed %}`，不要依赖 falsy 值渲染为空（升级后 `0` / `false` / `NaN` 会渲染出来）。完成标准：所有条件分支在"空串"与"0/false"两个用例下都验证过。
5. **跳过被拒的语法**：不要写 `{% include %}` / `{% render %}` / `{% layout %}`（保存即 400，文档未给替代方案）；跨模板复用只能在单模板内用 `assign` / `capture` / `{% for %}` 组织，或由调用侧拼接。完成标准：模板里不含这三类标签。
6. **转义要原样展示的标记**：正文里成对出现的 Liquid 示例必须包 `{% raw %}...{% endraw %}`；不成对的写法无需处理。完成标准：预览里示例按字面显示。
7. **走一次校验**：模板/活动靠保存时的 400（含行列号）兜底；若正文由外部系统生成并走 `/v1/send` 内联传入，则**没有这层校验**，必须先在己侧渲染/校验，或改成引用已保存的模板。完成标准：明确知道这条链路有没有校验，且坏 Liquid 不会静默发出（见第 8 步的口径）。
8. **用预览兜住静默失败**：渲染容错会让拼错的变量名/过滤器名完全不报错。完成标准：发送前用 `GET /templates/:id` + 指定联系人预览核对渲染结果；改前用 `GET /templates/:id/usage` 查影响面，需要变体就 `POST /templates/:id/duplicate`。
   - ⚠️ 文档口径：附件上限为"≤10 个 / 附件合计 ≤10MB / 整封消息 ≤40MB（SES 硬上限）"，前两项自托管可用 `MAX_ATTACHMENTS_COUNT`、`MAX_ATTACHMENT_SIZE_MB` 调整，40MB 不可调。数量或体积超限时具体错误码文档未说明——若被拒，先拆附件或缩体积再重试。
9. **定本地化策略**：短文案 → 单模板内 `{{locale}}` 分支；长邮件或多语言规模化 → 每语言一模板（`welcome-en` / `welcome-fr`），在代码或 workflow 里挑选。始终记住 Plunk 只翻译它自己控制的字符串（营销页脚、托管 unsubscribe/subscribe/preferences 页）。完成标准：明确列出"哪些由 Plunk 翻译、哪些自己出"，且 locale 用的是 14 种支持语言内的完整标签。
10. **不要试图自己写纯文本版本**：每封邮件（事务、活动、测试发送）都会由 Plunk 从 HTML 自动生成 text 替代件，API 没有 plain text 字段。完成标准：按四条保真规则自检——链接保留文字与目的地（`View invoice [https://...]`）、标题/段落/列表各占一行、布局表格被展平为散文、图片与追踪像素被丢弃；不要指望在 text 版本里塞与 HTML 不同的内容（不一致会触发它本想规避的同一批过滤器）。
11. **发送前跑一遍自检清单**：按顺序确认 ① `type` 与"是否尊重退订/是否要页脚"一致；② 需要退订出口时正文里有 `{{unsubscribeUrl}}` 或 `{{manageUrl}}`（HEADLESS 必填）；③ 所有变量在预览里都有值或可接受的空值；④ 条件分支在"空串"与"0/false"两个用例下验证过；⑤ 没有 `{% include %}` / `{% render %}` / `{% layout %}`；⑥ 需要字面展示的 Liquid 片段都包了 `{% raw %}`；⑦ 附件在三重上限内且已 base64 编码；⑧ 改动的模板先看过 `GET /templates/:id/usage` 的影响面。完成标准：八项全勾才能发；任何一项不确定就先做一次测试发送（`/v1/send`）或活动测试发送（见 `plunk-campaigns-analytics`）。
    - 若保存返回 400：照响应里的行列号改语法，回到第 5–7 步。
    - 若保存成功但预览里内容不对：按渲染容错三层定位（未知变量 / 未知过滤器 / 整体回退），回到第 3–4 步。
    - 若走 `/v1/send` 内联且内容不对：认定没有保存期校验，回到第 7 步改为己侧预渲染或引用已保存模板。

## B 边界与失效

1. **不要用 type 换投递率**：三种 type 在发送优先级上完全相同，为"送达率更高"选 TRANSACTIONAL 是错误的选型依据，而且会绕过退订、制造合规风险。
2. **模板问题永远不会让发送失败**：未知变量渲染为空、未知过滤器跳过、整体解析失败回退为纯替换。因此"发送成功"不能证明"内容正确"——内容正确性只能靠预览与测试发送。
3. **`/v1/send` 的内联正文不校验**：坏 Liquid 静默降级为纯 `{{variable}}` 替换并返回成功，条件/循环不生效且毫无错误信号。需要语法保障时必须走保存的模板。
4. **Liquid 升级有破坏性变更**：正文里平衡的标签会被执行（需 `{% raw %}`）、字段名含 `|` 或引号等运算符字符会被当表达式解析、`0/false/NaN` 不再渲染成空串。老模板必须按 `/guides/template-language` 的"Behaviour to check before upgrading"清单逐项复查。
   - ⚠️ 文档口径不一致，两者并列，不要替文档做取舍：`??` 兜底值的引号写法在 `/concepts/templates` 是 `{{firstName ?? 'there'}}`（带引号），在 `/guides/template-language` 是 `{{firstName ?? there}}`（不带引号），后者说明"带引号的兜底值不再泄漏引号"。两种写法都以你实际渲染结果为准，先预览再发。
5. **被拒的语法没有替代方案**：`{% include %}` / `{% render %}` / `{% layout %}` 一律拒绝，跨模板复用只能靠调用侧拼接或单模板内 `assign`/`capture`。
6. **本地化的边界是硬的**：Plunk 只翻译它控制的字符串（营销页脚与三个托管页面），你自己的 subject/body 永远不会被自动翻译；locale 不在 14 种支持语言内就静默回退到项目默认语言、再回退到英语。
7. **附件三重上限且编码有要求**：≤10 个附件、附件合计 ≤10MB、整封消息 ≤40MB，附件需 base64 编码放在 `attachments` 数组；超限的精确失败行为文档未说明（不要假定是 400）。
8. **纯文本版本不可控也没有字段可设**：无可读文本时（如纯图片邮件）Plunk 直接省略 text 部分而不是附一个空的——不要在这一侧做内容设计。
9. **高危/不可逆操作提示**：模板的 `from` 在创建与更新时都校验域名，未验证返回 403（不是 400/422），因此"先建模板后补域名验证"不可行；删除模板会影响所有引用它的 campaign 与 workflow 步骤，删前务必先 `GET /templates/:id/usage`。退订相关的不可逆动作见 `plunk-subscription-consent`。
10. **dashboard-only 边界**：模板预览（选联系人看渲染结果）、拖拽编辑器的 `style` 元数据等只能在 dashboard 完成，API 侧没有等价端点；通过 API 创建时可忽略 `style`。
