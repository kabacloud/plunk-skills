# 事件追踪

## R 原文依据

- [/api-reference/public-api/trackEvent](https://docs.useplunk.com/api-reference/public-api/trackEvent)："Track an event for a contact. Automatically creates or upserts the contact, then records the event. Tracked events can be used as workflow triggers, segment filters, and audience filters."
- [/api-reference/public-api/trackEvent](https://docs.useplunk.com/api-reference/public-api/trackEvent)："Reserved event names (rejected with `VALIDATION_ERROR` and code `reserved_event`): anything matching `email.*`, `contact.subscribed`, `contact.unsubscribed`, `segment.<slug>.entry`, `segment.<slug>.exit`. These are emitted by Plunk itself."
- [/api-reference/public-api/trackEvent](https://docs.useplunk.com/api-reference/public-api/trackEvent)："**New** contacts default to subscribed (`true`). **Existing** contacts keep their current state unless you pass an explicit value here. Pass `false` to track an event without resubscribing an unsubscribed contact."
- [/api-reference/public-api/trackEvent](https://docs.useplunk.com/api-reference/public-api/trackEvent)："Idempotency: re-tracking the same event creates a new event record. Send an `Idempotency-Key` header to have a repeated request refused with `409` instead."
- [/api-reference/overview](https://docs.useplunk.com/api-reference/overview)："`POST /v1/track` — Track an event for a contact. Auto-creates or upserts the contact. Triggers workflows. Key: `pk_*` or `sk_*`."
- [/guides/tracking](https://docs.useplunk.com/guides/tracking)："Enabled: Tracking is applied to every email. Disabled: No tracking is applied to any email. Marketing only: Tracking is applied to marketing sends... Transactional sends — anything via `/v1/send` using a transactional template, or campaigns of type TRANSACTIONAL — are sent without tracking."
- [/guides/tracking](https://docs.useplunk.com/guides/tracking)："Tracking is **all or nothing** within a single send — there's no separate toggle for opens vs clicks. Pick the mode that fits your privacy and analytics requirements."
- [/guides/tracking](https://docs.useplunk.com/guides/tracking)："Some clients (and corporate mail security gateways) pre-fetch images, which can produce false positive opens... Apple Mail Privacy Protection masks individual opens — opens from Apple Mail are recorded but reflect proxy fetches rather than human reads."

## I 解释

`/v1/track` 是把"你的系统里发生的事"搬进 Plunk 的唯一入口，也是唯一可以用 public key（`pk_`）调用的端点——因为它的典型调用方是浏览器与移动端。它做两件事且顺序固定：先按 email 自动创建或 upsert 联系人，再在该联系人上记录事件。事件随后可被用作 workflow 触发、segment 过滤与受众过滤，也就是"业务行为 → 自动化"的接线点。

第一层是命名约束。Plunk 自己会发出一批系统事件，因此这些名字**不能手动上报**：任何匹配 `email.*` 的名字，以及 `contact.subscribed`、`contact.unsubscribed`、`segment.<slug>.entry`、`segment.<slug>.exit`。命中会以 `VALIDATION_ERROR` + 错误码 `reserved_event` 被拒。从其他平台迁移事件命名时最容易踩这个坑（例如为了"补数据"自己去 track `email.opened`）——这些事件只能监听，不能伪造。

第二层是重复与幂等。重复 track 同一事件会产生**新的**事件记录，不会自动去重；只有带 `Idempotency-Key` 时，重复请求才会被 409 拒绝。所以幂等是调用方的选择，不是端点的默认行为。

第三层是订阅状态的副作用。新联系人由 `/v1/track` 创建时默认 `subscribed: true`；已有联系人**保持原状态**，除非显式传 `subscribed`。这条规则的实际用途是：给已退订的人打事件又不想把他重新订阅时，显式传 `false`。把"打点"和"改订阅状态"当成一件事，是这里最常见的误用。

第四层是邮件侧的追踪（open/click），它与 `/v1/track` 是两套东西：前者在 Settings → Tracking 按项目三选一，且**单封邮件内全有或全无**（不能只开点击不开打开）。Enabled = 所有邮件都追踪；Disabled = 任何邮件都不追踪；Marketing only = 只追踪营销类发送（campaign，以及使用 marketing 或 headless 模板的 workflow 步骤），事务类（`/v1/send` 走 transactional 模板，或 TRANSACTIONAL campaign）不加追踪。这里的关键认知是"边界由模板/活动类型决定，与端点无关"——所以想让事务邮件不加追踪，正确做法是改模板类型或模式，而不是改调用端点。

第五层是数据可信度。打开依赖客户端加载远程图片（禁用图片就没有打开记录），企业安全网关预取会造成假打开，Apple Mail Privacy Protection 把真实打开掩盖成代理抓取；点击则会被安全工具预扫描污染。因此"未打开"不等于"不感兴趣"，用 `email.open` 驱动"未打开就补发"这类自动化会误触发。

第六层是 `data` 的写入语义——它决定了"模板里能不能用到这个值"。`data` 是 patch 语义：只改你传的键。持久化值（原语、普通对象）会存到联系人上并成为模板变量；一次性值（订单号、交易明细）要写成 `{ value, persistent: false }`，只参与本次渲染、不落库；`null` 删除该字段；空串被忽略（既不存也不覆盖）；保留键会被静默过滤（发了既不存储也不报错）。此外，`data` 对 dashboard 与所有有 API 权限的人可见，所以密钥类信息不该走这条通道，一次性凭据用非持久化值。

把上述约束摊开，就是一张调用前自检表：

| 检查项 | 规则 | 违反后的表现 |
| --- | --- | --- |
| 事件名 | 不得匹配 `email.*`、`contact.subscribed`、`contact.unsubscribed`、`segment.<slug>.entry/exit` | 被拒，错误码 `reserved_event`（状态码文档未统一，见 A1 示例 2） |
| 密钥 | `pk_` 只能调 `/v1/track`；其余端点用 `sk_` | 用 `pk_` 调发信端点返回 401 |
| 幂等 | 重复 track 会产生新记录，只有 `Idempotency-Key` 能以 409 拒绝重复 | 事件被重复计数、workflow 被重复触发 |
| 订阅副作用 | 新联系人默认 `true`；已有联系人保持原状除非显式传值 | 打点顺手把已退订者重新订阅 |
| 追踪模式 | 项目级三选一，单封邮件内全有或全无 | 想只开点击却做不到；自托管缺配置集时 Marketing only 不可选 |

## A1 示例

**示例 1 · 后端上报注册事件**（来自 `/api-reference/public-api/trackEvent` 与 recipes）

- 输入/问题：用户提交表单后要在 Plunk 里留下可触发 welcome 流程的事件，并把名字带进去。
- 怎么执行：服务端用 `sk_` 调 `POST /v1/track`，body 为 `{ "event": "waitlist.joined", "email": "ada@example.com", "data": { "firstName": "Ada", "product": "Beta" } }`；浏览器侧若确实要打点，只能用 `pk_` 且只能打 `/v1/track`。
- 产出/预期输出：联系人被 upsert（新联系人默认 subscribed），事件被记录，`data` 里的键成为模板变量 `{{firstName}}` / `{{product}}`。文档未给出响应体的完整字段清单（只给 200 Event tracked successfully）。

**示例 2 · 用保留名"补数据"被拒**（来自 counter-example e58）

- 输入/问题：迁移历史事件时手工 track `email.opened` / `contact.unsubscribed` 来补数据或触发 workflow。
- 怎么执行：这些名字属于 Plunk 自己发出的系统事件，只能监听；自定义事件名必须避开 `email.*` 前缀等保留模式。
- 产出/预期输出：请求被拒，错误为 `VALIDATION_ERROR` 且 `code: reserved_event`（该 counter-example 记为 422）。⚠️ 文档口径不一致，两者并列，不要替文档做取舍：trackEvent 页只写"rejected with `VALIDATION_ERROR` and code `reserved_event`"未给状态码，counter-example 记为 422；排障时以错误码 `reserved_event` 为准，不要只匹配状态码。

**示例 3 · 给已退订者打事件但不重新订阅**（合成演练，按 trackEvent 页的 `subscribed` 语义可复算）

- 输入/问题：一个已退订用户产生了订单事件，需要记录事件用于后续对账，但不能把他重新拉回营销列表。
- 怎么执行：`POST /v1/track` 时显式传 `subscribed: false`；若不传，已有联系人保持原状态（不会自动变回订阅），但新建联系人会默认 `true`。这条规则还有第二个用途：想借打点把某人重新订阅时，显式传 `subscribed: true`。
- 产出/预期输出：事件被记录且订阅状态可控。⚠️ 注意"新建"与"已有"两条路径默认值不同：新联系人由 `/v1/track` 创建即默认订阅，这与 `POST /contacts` 的默认 false 不一致（见 `plunk-contacts-and-fields`）。

**示例 4 · 自托管想让事务邮件不带追踪**（来自 counter-example e62）

- 输入/问题：自托管实例上要区分事务邮件与营销邮件的追踪，但在 Settings → Tracking 里找不到"Marketing only"选项。
- 怎么执行：先在 SES 里建第二个不含追踪的配置集（仅 Sends/Deliveries/Bounces/Complaints），再设置 `SES_CONFIGURATION_SET_NO_TRACKING`；不设该变量时该模式选项会被隐藏。
- 产出/预期输出：Marketing only 模式可选并生效——事务类发送不再被加追踪像素。若不配，模式退化为全开或全关，事务邮件也会被追踪。

**示例 5 · 重复上报被算两遍**（合成演练，按 trackEvent 页的幂等语义可复算）

- 输入/问题：订单系统的重试机制把同一次"已发货"事件上报了三次，workflow 触发了三次欢迎/通知流程。
- 怎么执行：重复 track 同一事件会产生**新的事件记录**，端点不会自动去重。为每个逻辑操作生成稳定的 `Idempotency-Key`（如 `order-1234-shipped`），重复请求会被 409 拒绝——把 409 当作"已处理"而不是错误。
- 产出/预期输出：重试路径上第二次及以后的请求返回 409，事件只记录一次、workflow 只触发一次。

## A2 何时会用

**用户场景**

1. 要把应用里的业务行为（注册、下单、升级、取消）接进 Plunk，用来触发 workflow 或更新 segment。
2. track 请求被拒，错误码是 `reserved_event`，或不确定事件名能不能用。
3. 担心重复上报会把事件算两遍，需要幂等方案。
4. 打点时不小心把已退订的人重新订阅了，或反过来想让某人保持订阅状态。
5. 要决定项目的打开/点击追踪模式（Enabled / Disabled / Marketing only），尤其是事务邮件要不要追踪。
6. 打开率/点击率看起来异常高，或想用"未打开"做再营销触发。

**语言信号（中英双写）**

`事件追踪` / `track event` / `/v1/track` / `event name` / `reserved_event` / `VALIDATION_ERROR` / `pk_` / `public key` / `Idempotency-Key` / `open tracking` / `click tracking` / `tracking mode` / `marketing only` / `false positive opens` / `Apple Mail Privacy Protection` / `pre-fetch` / `email.open` / `email.click` / `configuration set`

**一句话判据**：凡是"要把业务行为记到某个邮箱上"就是 `/v1/track`；凡是"要判断邮件有没有被看到/点过"就是追踪模式与互动事件——两者不要混用，也不要试图用前者伪造后者。

**与相邻能力的区分**

- 本卡负责：`/v1/track` 的上报契约（命名、幂等、订阅副作用）与项目级打开/点击追踪模式的选择及可信度边界。
- `plunk-contacts-and-fields` 负责联系人本身、`data.*` 字段与保留键；本卡只说明 track 会 upsert 联系人并写入 `data`。
- `plunk-workflows` 负责事件如何驱动流程（triggerType EVENT、WAIT_FOR_EVENT）；本卡只负责让事件正确落库。
- `plunk-campaigns-analytics` 负责活动统计里怎么读这些互动数字；本卡只负责这些数字本身的失真来源。
- `plunk-self-hosting` 负责 `SES_CONFIGURATION_SET_NO_TRACKING` 等环境变量的完整部署；本卡只在"Marketing only 不可用"这一处指向它。
- `plunk-subscription-consent` 负责订阅状态与退订语义本身；本卡只处理"打点时的 subscribed 副作用"。

**按诉求快速分流**：

- "要把业务行为打点进 Plunk / track 被拒" → 本卡（上报契约）。
- "要决定打开点击追踪模式、数据不可信" → 本卡（模式与可信度）。
- "事件要怎么触发流程、怎么等人" → `plunk-workflows`。
- "联系人字段与保留键" → `plunk-contacts-and-fields`。
- "活动里的打开点击怎么读" → `plunk-campaigns-analytics`。

## E 执行步骤

**输入契约**

必需信息：
1. 事件名（`event`）与收件人邮箱（`email`）——两者都是必填。
2. 要一并写入的 `data`（哪些要持久化、哪些是一次性值）。
3. 调用位置：浏览器/移动端（只能用 `pk_`）还是服务端（用 `sk_`）。
4. 是否需要幂等（同一逻辑操作会不会重复上报）。
5. 这次打点是否允许改变订阅状态（决定要不要显式传 `subscribed`）。

缺失时先问：事件名是否会命中保留模式、同一次业务操作是否可能重复触发、接收方是否可能已退订。

**输出契约**

产出：可直接提交的 `POST /v1/track` 请求体（含正确的 event 名、data 与必要的 `subscribed`）、幂等键策略、以及项目追踪模式的结论。验收：① 请求返回 200 且事件出现在联系人动态里；② 重复请求带同一 Idempotency-Key 时被 409 拒绝；③ 联系人的订阅状态与预期一致（没有被意外改订阅或退订）。

1. **先查事件名是否保留**：避开 `email.*`、`contact.subscribed`、`contact.unsubscribed`、`segment.<slug>.entry`、`segment.<slug>.exit`。完成标准：事件名不匹配任何保留模式；若被拒，确认错误码是 `reserved_event` 而不是"名字写错了"。
2. **选密钥与调用位置**：浏览器/移动端只能用 `pk_`（且只有 `/v1/track` 可用）；服务端用 `sk_`。完成标准：前端代码里不出现 `sk_`（泄露密钥），服务端不用 `pk_` 调别的端点（会 401）。
3. **处理订阅副作用**：新联系人默认 `subscribed: true`；已有联系人保持原状态除非显式传值。给已退订者打点又不想重新订阅 → 显式传 `subscribed: false`。完成标准：打点前后各核对一次订阅状态，确认没有意外翻转。
4. **决定 data 里哪些要落库**：持久化的原语/普通对象会成为联系人字段与模板变量；一次性值（订单号、交易明细）用 `{ value, persistent: false }`；`null` 删除字段，空串被忽略。完成标准：`data` 里不含应保密的信息（dashboard 与有 API 权限的人都能看到），一次性凭据用非持久化值。
5. **需要去重就加幂等键**：重复 track 同一事件会产生新记录；带 `Idempotency-Key` 时重复请求返回 409。完成标准：为每个逻辑操作生成稳定键（如 `order-1234-shipped`），并接受 409 作为"已处理"而非错误。
6. **定项目追踪模式**（Settings → Tracking）：所有邮件都要互动数据 → Enabled；完全不要追踪（隐私优先）→ Disabled；只要营销类有、事务邮件不带追踪 → Marketing only。完成标准：明确接受"单封邮件内追踪是全有或全无"，不能只开点击或只开打开。
7. **若选 Marketing only，确认平台前提**：托管版直接可选；自托管必须先配 no-tracking configuration set 并设置 `SES_CONFIGURATION_SET_NO_TRACKING`，否则选项被隐藏。完成标准：模式可见且事务类发送（`/v1/send` + transactional 模板，或 TRANSACTIONAL campaign）确实不带追踪。
8. **选对"互动信号"的强度**：从弱到强依次是 打开（失真最多）→ 点击（会被预扫描污染，但仍有意图）→ 自定义事件（本卡上报的业务行为）→ 自定义字段（明确的属性）。完成标准：对外报互动率时标注口径；驱动自动化时优先用点击及以上，凡是以 `email.open` 为唯一判据的自动化（如"未打开就补发"）都要改成更强的信号或加人工阈值。
    - ⚠️ 文档口径不一致，两者并列，不要替文档做取舍：打开/点击追踪存在**系统性假阳性**（预取与预扫描产生假打开/假点击，Apple Mail Privacy Protection 掩盖真实打开），文档没有给出剔除或校准方法；需要更可信的互动口径只能自行换用更强的信号并验证。
9. **不要指望用 track 伪造系统事件**：`email.*`、`contact.subscribed/contact.unsubscribed`、`segment.<slug>.entry/exit` 都只能由 Plunk 发出。需要"某人打开了邮件"这类数据时，正确做法是打开/点击追踪（模式与可信度见第 6–8 步），而不是手动上报一个 `email.opened`。完成标准：所有自定义事件名都是业务语义（如 `order.shipped`），且不与保留模式冲突。
    - 若上报被拒且错误码是 `reserved_event`：回到第 1 步改名（不要用状态码作为唯一判据）。
    - 若重复上报已被记录：回到第 5 步补 `Idempotency-Key`。
    - 若自托管选不到 Marketing only：回到第 7 步先配 no-tracking configuration set。
    - 若数据看起来虚高：回到第 8 步换用更强的信号，不要试图"清洗"打开数据。

## B 边界与失效

1. **保留事件名不能上报**：命中 `email.*` 等模式会被拒（错误码 `reserved_event`），事件不落库、workflow 不触发。这些是 Plunk 自己发出的系统事件，只用于监听。
2. **重复上报不会自动去重**：同一事件重发会产生新记录，只有 `Idempotency-Key` 能以 409 拒绝重复；不要假设端点自带幂等。
3. **打点可能顺带改变订阅状态**：新联系人默认订阅。给已退订者打点时不显式传 `subscribed: false`，就有把人重新拉回营销列表的风险（而新建联系人恰恰又默认订阅）。
4. **追踪没有细粒度开关**：单封邮件内是全有或全无，不能只开点击不开打开，也不能对单个收件人例外。需要部分追踪只能靠模式（Marketing only）间接实现。
5. **Marketing only 有平台前提**：自托管未配 no-tracking configuration set 时该模式不可用，会退化为全开或全关，事务邮件也被加追踪像素。
6. **互动数字带系统性假阳性**：预取、预扫描与 Apple MPP 让打开/点击失真。用 `email.open` 做"未打开就补发"会误触发，对外报 KPI 会虚高。
7. **不要把敏感信息放进 data**：`data` 会落在联系人上且对 dashboard 与有 API 权限的人可见；一次性凭据应使用 `{ value, persistent: false }`，密钥类信息不要走这条通道。
8. **前端只能打这一个端点**：`pk_` 仅限 `/v1/track`，用它调 `/v1/send` 等端点会 401；为绕过而把 `sk_` 下发到前端会造成密钥泄露（处置见 `plunk-api-essentials`）。
9. **本卡不负责事件如何驱动流程**：workflow 的触发类型、等待事件、条件分支都在 `plunk-workflows`；本卡只保证事件正确到达。
10. **dashboard-only 边界**：Settings → Tracking 的模式切换与联系人动态里的事件查看主要在 dashboard 完成，API 侧没有等价的"设置追踪模式"端点；本卡的 `/v1/track` 只覆盖上报侧。
