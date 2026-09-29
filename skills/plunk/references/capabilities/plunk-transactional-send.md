# 事务邮件发送

## R 原文依据

- [/api-reference/public-api/sendEmail](https://docs.useplunk.com/api-reference/public-api/sendEmail)："**Required content:** either a `template` ID, **or** both `subject` and `body`. Template fields can be overridden by explicit request fields."
- [/api-reference/public-api/sendEmail](https://docs.useplunk.com/api-reference/public-api/sendEmail)："**Sender:** `from` is required unless using a template that already has a `from` configured. The sender's domain must be verified."
- [/api-reference/public-api/sendEmail](https://docs.useplunk.com/api-reference/public-api/sendEmail)："**Multiple recipients:** when `to` is an array, each recipient is processed sequentially with its own contact upsert and rendered email — there is no batch-send semantics. Sending is always immediate; for scheduled sends, use a Campaign."
- [/concepts/transactional-emails](https://docs.useplunk.com/concepts/transactional-emails)："you can include up to 10 attachments per email with a maximum total size of 10MB. Attachments should be base64 encoded... The total message size cannot exceed 40 MB."
- [/api-reference/public-api/sendEmail](https://docs.useplunk.com/api-reference/public-api/sendEmail)："`subscribed`: Subscription state to apply to the recipient. For **new** contacts, defaults to `false` on `/v1/send`. For **existing** contacts, omitting this preserves their current state — pass `true` or `false` to explicitly change it."
- [/api-reference/public-api/sendEmail](https://docs.useplunk.com/api-reference/public-api/sendEmail)：200 响应里 `data.emails[].email` —— "Plunk email record ID. Use this to correlate webhook events (which include this ID as 'emailId' in the event data) with your send requests."
- [/api-reference/public-api/sendEmail](https://docs.useplunk.com/api-reference/public-api/sendEmail)：`### 200 — Email queued successfully` 的示例体写的是 `{ "success": false, "data": { "emails": [...] } }`（文档生成瑕疵，见 B 段）

## I 解释

Plunk 有三条发送通道，`/v1/send` 是其中唯一"由你的系统即时触发、一对一"的那条。选型三问是：触发方式（系统调用 vs 人工排程 vs 事件驱动）、收件人数量（一对一 vs 一对多）、时间形状（立即 vs 排程 vs 多步骤等待）。一对一 + 系统即时触发 → `/v1/send`；一对多 + 一次性 + 要排程 → Campaign；多步骤 + 事件驱动 + 要等待或分支 → Workflow。三者共用同一套 Template 与同一套订阅规则，所以通道选择不改变"能否送达"的判定，只改变编排形态。

`/v1/send` 的请求形状只有一个核心约束：内容不能既没有也没有。要么给 `template` ID，要么 `subject` + `body` 齐备。给了 `template` 时，模板会提供 subject / body / from / reply-to，但请求里显式传的同名字段会覆盖模板值——这是"一处改模板、一处改请求"时最需要小心的优先级。

`from` 的门槛落在域名验证上：引用了已配 `from` 的模板时可以省略，否则必填，且域名必须已验证，否则 403。

响应侧真正有价值的只有 `data.emails[].email`（即 `emailId`）。它是 Plunk 的邮件记录 ID，所有 email 事件的 webhook 载荷里都带同一个值，也作为退订链接上的 `?e=` 参数。有了它，你不必靠"邮箱 + 时间戳"去模糊匹配 webhook，也不必为了拿 provider 的 `messageId` 而额外监听 `email.sent`。

两个反直觉的点值得单独记住：多收件人不是批量语义——`to` 传数组时逐个 upsert 联系人、逐个渲染，变量按各自联系人解析；订阅默认值在三个端点上并不一致，`/v1/send` 的新联系人默认 `subscribed=false`，而 `/v1/track` 默认 `true`。

## A1 示例

**示例 1 · 一次最小调用与三种等价写法**（来自 `/api-reference/overview` 与 `/api-reference/public-api/sendEmail`）

- 输入/问题：后端要发一封密码重置邮件，收件人 `user@example.com`。
- 怎么执行：`POST https://next-api.useplunk.com/v1/send`，头 `Authorization: Bearer $PLUNK_SECRET_KEY` + `Content-Type: application/json`，body `{"to":"user@example.com","subject":"Hello","body":"<p>Your message here</p>"}`。
- 产出/预期输出：200 + `{success: true, data: {emails: [{contact: {id, email}, email: "<emailId>"}]}}`。注意文档给出的示例体里写的是 `"success": false`（瑕疵，见 B 段）。

**示例 2 · 用 template 并只覆盖 subject**（来自 `/api-reference/public-api/sendEmail`）

- 输入/问题：团队维护了一份带 `from` 与 reply-to 的模板，但这次发送要换一行主题。
- 怎么执行：`{"to":"user@example.com","template":"<uuid>","subject":"本次专属主题","data":{"resetUrl":"..."}}`。`from` 省略（模板已配）；`subject` 显式传入，覆盖模板的 subject；`data` 里的值喂模板变量。
- 产出/预期输出：渲染后的邮件用新主题、模板的 from 与 reply-to、模板 body 中 `{{resetUrl}}` 被替换。文档未给出覆盖后 body 变量解析失败的报错行为。

**示例 3 · 多收件人逐个渲染**（按 `/api-reference/public-api/sendEmail` 规则可复算的合成演练）

- 输入/问题：给 3 个客户各发一份含其名字的收据。
- 怎么执行：`to` 传 3 个元素的数组，`data` 里放各人字段——但数组是顺序处理、每个收件人各自 upsert 并各自渲染，因此变量按各自联系人解析。若需要按收件人幂等控制，必须一人一请求一键（见 `plunk-api-essentials`）。
- 产出/预期输出：3 条独立渲染的邮件 + 3 个独立 `emailId`；中途 5xx 时部分人可能已收到。

**示例 4 · 带 PDF 附件与内联 Logo**（按 `/api-reference/public-api/sendEmail` 字段约束可复算的合成演练）

- 输入/问题：订单确认邮件要附一张发票 PDF，并在正文顶部嵌入品牌 Logo。
- 怎么执行：`attachments` 放两项——发票项 `{filename: "invoice-1234.pdf", content: "<base64>", contentType: "application/pdf"}`（`disposition` 默认 `attachment`）；Logo 项额外给 `contentId: "logo"` 与 `disposition: "inline"`，正文用 `<img src="cid:logo">` 引用。
- 产出/预期输出：一封含下载型附件与内嵌图片的邮件。约束是数量 ≤10、附件合计 ≤10MB、整条消息 ≤40MB；`filename` 不能含换行或引号字符。文档未说明超限时的具体错误码（未说明，按 422 `VALIDATION_ERROR` 处理即可，字段路径形如 `attachments.0.filename`）。

> 订阅默认值的三点不一致（`/v1/send` 新联系人默认 `false`，`/v1/track` 默认 `true`，`POST /contacts` 用传入值）与"省略即保持"的语义，见 E 段第 4 步与 B 段。

## A2 何时会用

**用户场景**

1. 系统要在用户重置密码、下单、触发告警时立刻发一封一对一邮件。
2. 要在发信时顺带写入或更新联系人 `data.*`（如一次性验证码、订单号）。
3. 要给邮件加附件（发票 PDF、内联 Logo）并想知道容量上限。
4. 发出的邮件需要能和业务记录对上（用 `emailId` 关联 webhook）。
5. 纠结该用 `/v1/send`、Campaign 还是 Workflow，或者想定时发一对一邮件。

**语言信号（中英双写）**

`事务邮件` / `transactional email` / `发送邮件` / `send email` / `/v1/send` / `密码重置` / `password reset` / `order confirmation` / `emailId` / `attachments` / `附件` / `base64` / `template` / `subscribed` / `inline content` / `多收件人` / `multiple recipients`

**与相邻能力的区分**

- 本卡负责：`POST /v1/send` 这一个端点的字段契约、必填组合、容量上限与回执用法。
- `plunk-templates-liquid` 负责模板本身怎么写、Liquid 语法与变量解析，本卡只在"请求字段覆盖模板字段"这一处交界。
- `plunk-subscription-consent` 负责订阅状态与退订语义：MARKETING 模板发给已退订联系人会被跳过，这是模板类型而非通道决定的。
- `plunk-api-essentials` 负责认证、幂等键、错误码与限流等跨端点契约。
- `plunk-campaigns-analytics` 负责一对多与排程发送。
- `plunk-workflows` 负责多步骤、事件驱动的编排。

## E 执行步骤

**输入契约**

必需信息：
1. 收件人（`to`：字符串 / `{name, email}` / 二者的数组）。
2. 内容来源：`template` ID，还是 `subject` + `body`（HTML）。
3. 发件人 `from`（字符串或 `{name, email}`），或确认所用模板已配好 `from`。
4. 是否要改订阅状态、是否要带附件、是否需要事后用 `emailId` 关联 webhook。

缺失时先问：是立即发还是要定时（定时必须转 Campaign）、收件人是否可能已退订、模板是否已存在于本项目的 ID、发信域名是否已验证。

**请求字段速查**

| 字段 | 必填 | 约束 / 语义 |
| --- | --- | --- |
| `to` | 必填 | 字符串 / `{name,email}` / 数组；数组为逐个处理、无批量语义 |
| `template` | 内容二选一 | uuid；提供 subject / body / from / reply-to |
| `subject` | 内容二选一 | 无 template 时与 `body` 齐备；不能含换行字符 |
| `body` | 内容二选一 | HTML；Plunk 自动生成并同时发送纯文本版本 |
| `from` | 模板已配 `from` 时可省 | 字符串或 `{name,email}`；域名必须已验证，否则 403 |
| `name` | — | **已废弃**，仅在 `from` 是字符串且未带 name 时回退 |
| `reply` | 可选 | reply-to 地址 |
| `subscribed` | 可选 | 新联系人默认 `false`；已存在联系人省略即保持 |
| `data` | 可选 | 原语 / `null` 删除字段 / `""` 跳过 / `{value, persistent:false}` 不落库 |
| `headers` | 可选 | 名禁 CRLF；值禁 CRLF 且 ≤998 字符 |
| `attachments[]` | 可选 | `filename` / `content`(base64) / `contentType` 三必填 |

**三条发送通道选型对照**

| 判据 | `/v1/send` | Campaign | Workflow |
| --- | --- | --- | --- |
| 触发方式 | 你的系统即时调用 | 人工创建或排程 | 事件驱动、自动 |
| 收件人 | 一对一 | 一对多（ALL / SEGMENT / FILTERED） | 一对一或多步骤群体 |
| 时间形状 | 永远立即 | 可 `scheduledFor` 排程 | 可 DELAY / WAIT_FOR_EVENT |
| 状态机 | 无 | DRAFT / SCHEDULED / SENDING / SENT / CANCELLED | WorkflowExecution 状态 |
| 典型场景 | 密码重置、订单确认、告警通知 | 群发、周报、公告 | 欢迎序列、弃购挽回 |
| 共用 | 同一套 Template 与同一套订阅规则——通道选择不改变送达判定 | 同左 | 同左 |

**步骤**

1. **先做通道选型**：一对一 + 系统即时触发 → `/v1/send`；一对多 + 要排程 → Campaign；多步骤 + 事件驱动 → Workflow。完成标准：明确说出触发方式、收件人数量、时间形状三问的答案。**若答案是"要定时"的一对一 → 转 Campaign，本端点做不到**（`/v1/send` 永远立即发送）。
2. **确定 `from`**：引用了已配 `from` 的模板 → 可省略；否则必填字符串或 `{name, email}`（旧字段 `name` 已 deprecated，仅在 `from` 是字符串且未带 name 时作为回退）。完成标准：`from` 的域名在本项目已验证——未验证会 403，此时转 `plunk-domain-deliverability`。
3. **确定内容组合**：给了 `template` → 模板提供 subject / body / from / reply-to，请求里显式传 `subject` / `body` / `from` / `reply` 会覆盖模板值；不给 `template` → `subject` 与 `body` 必须齐备。完成标准：二选一且只有一条路径被选中，不存在"既没 template 又只有 subject"的请求。
4. **处理订阅语义**：不想改状态 → **省略** `subscribed`；想改 → 显式传 `true`/`false`。记住三个端点默认值不同：`/v1/send` 新联系人默认 `false`，`/v1/track` 默认 `true`，`POST /contacts` 用传入值（默认 `false`）；对已存在的联系人三者一致——省略即保持。完成标准：double opt-in 场景两次调用都显式传 `false`；不存在"发一封邮件意外把人订阅上"的路径。
5. **填 `data`（可选）**：值可为原语（存到联系人并作为模板变量）、`null`（删除该字段）、空字符串（跳过、不覆盖已有值）、`{ value, persistent: false }`（只参与本次渲染、不落库，适合一次性验证码与 magic link）。保留键 `id` / `plunk_id` / `plunk_email` / `email` / `unsubscribeUrl` / `subscribeUrl` / `manageUrl` 会被静默过滤。完成标准：一次性敏感值用了 `persistent: false`，业务字段没有撞保留键。
6. **加附件（可选）**：`attachments[]` 每项三必填 `filename` / `content`（base64）/ `contentType`（MIME）；内联图片额外配 `contentId` + `disposition: "inline"`，正文用 `<img src="cid:yourContentId">` 引用。完成标准：数量 ≤10、附件合计 ≤10MB、整条消息 ≤40MB（40MB 是 SES 硬上限，自托管也不可调；前两项自托管可用 `MAX_ATTACHMENTS_COUNT` / `MAX_ATTACHMENT_SIZE_MB` 调）。
7. **自定义邮件头（可选）**：`headers` 的 header 名不能含 `\r\n`，值不能含 `\r\n` 且 ≤998 字符；`subject` 不能含换行字符。完成标准：所有动态拼接的 subject / header 都做过换行与长度校验（超限或含 CRLF 会被当作 header injection 拒绝）。
8. **多收件人决定是否拆分**：`to` 传数组是顺序处理、每个收件人各自 upsert 与渲染，没有批量语义。需要按收件人幂等或精准重试 → 拆成一人一请求、各自带独立 `Idempotency-Key`（见 `plunk-api-essentials`）。完成标准：不存在"一个请求带一批收件人且需要部分重试"的场景。
9. **发出并取 `emailId`**：`POST /v1/send`，200 的 `data.emails[].email` 即 Plunk 邮件记录 ID（形如 UUID）。完成标准：在首次成功响应里把 `emailId` 落库——它是后续 webhook 事件（载荷里同名 `emailId`）与业务记录对齐的唯一可靠锚点，也是退订链接上的 `?e=` 参数值。
10. **⚠️ 文档口径不一致，两者并列，不要替文档做取舍**：多个端点的 2xx 示例体里硬编码了 `"success": false`（`/v1/send` 的 200 示例即如此），与标题 "Email queued successfully" 及 errors 页"success 对成功请求恒为 true"矛盾；另有 `POST /contacts` 的 200/201 示例里 `_meta.isNew` 与 `isUpdate` 同时为 false。契约以**字段说明文字**为准、不以示例为准：客户端应判 HTTP 状态码与 `data` 结构，不要因为看到 `success: false` 就把成功当失败。

**输出契约**

产出：
1. 一份可直接发出的请求体（含 `to` / 内容来源 / `from` 三要素齐全）。
2. 落库的 `emailId`（以及每个收件人对应的 contact id）。
3. 若涉及订阅：预期的订阅状态结论（是否会产生 `contact.subscribed` / `contact.unsubscribed` 事件）。

验收：
- 请求能通过 422 schema 校验（否则 `error.errors` 会给出 `attachments.0.filename` 这类点路径字段）；
- `from` 域名已验证，无 403；
- 200 响应的 `data.emails` 条数与收件人数一致；
- `emailId` 已写入业务表，且后续 webhook 能用它对齐。

## B 边界与失效

1. **不要用 `/v1/send` 做定时发送**：它永远立即发送。要排程的一对一邮件只能转用 Campaign。
2. **不要把多收件人当批量**：`to` 数组是顺序逐个处理，中途 5xx 时部分收件人可能已收到；需要精准控制就一人一请求一键。
3. **不要以为发到了就等于对方收到**：MARKETING / HEADLESS 模板会尊重 opt-out，已退订联系人会被跳过——这是模板类型决定的，不是通道决定的（见 `plunk-subscription-consent`）。必须触达已退订的人时用 TRANSACTIONAL 模板。
4. **不要给 `subject` / `headers` 拼未校验的动态内容**：换行与 CRLF 会被当作 header injection 拒绝；header 值还有 998 字符上限。
5. **没有纯文本字段**：`body` 填 HTML，Plunk 自动生成并同时发送纯文本版本以规避 HTML-only 的垃圾惩罚。不要去找一个不存在的 `text` 字段。
6. **inline subject/body 不做 Liquid 预校验**：解析不了的部分会回退成纯 `{{variable}}` 替换而不报错。模板内语法问题不会在这里暴露，需要单独验模板（见 `plunk-templates-liquid`）。
7. **附件三重上限中 40MB 不可调**：数量与单封附件总量自托管可调，整条消息 40MB 是 SES 硬上限。
8. **⚠️ 文档口径不一致，两者并列，不要替文档做取舍**：成功响应示例里的 `"success": false` 是 OpenAPI 生成的瑕疵，契约以字段说明文字与 errors 页为准，不要照示例断言。
9. **模板类型不改变通道职责**：`MARKETING` 模板经 `/v1/send` 发出仍然尊重退订；想绕开订阅状态必须换 TRANSACTIONAL 模板，而不是换端点。
10. **本卡不含发送后的健康度判断**：退信、投诉与列表卫生见 `plunk-sending-health`；打开/点击的统计口径与假阳性见 `plunk-campaigns-analytics`。
