# 分群与过滤器 DSL

## R 原文依据

- [/guides/segment-filters](https://docs.useplunk.com/guides/segment-filters)：「The same filter format powers three things: **dynamic segments**, **campaign audiences** with `audienceType: "FILTERED"`, and **`CONDITION` steps inside workflows**. Learn it once, use it everywhere.」
- [/guides/segment-filters](https://docs.useplunk.com/guides/segment-filters)：「A filter has two parts: a top-level connector (`AND` or `OR`) and a list of **groups**. Each group holds one or more conditions, which are combined with `AND` inside the group... Groups can nest by setting `conditions` on a group.」
- [/guides/segment-filters](https://docs.useplunk.com/guides/segment-filters)：字段命名空间「`email`；`subscribed`；`createdAt`, `updatedAt`；`data.<path>` — supports nested paths；`event.<eventName>`；`email.<activity>` — sent, delivered, opened, clicked, bounced, complained；`segment.<segmentId>`」
- [/guides/segment-filters](https://docs.useplunk.com/guides/segment-filters)：「`equals` — Exact match. Case-insensitive on `email`, exact on `data.*`. `contains` — Case-insensitive substring match.」；「If the value is just a date (`YYYY-MM-DD`), the comparison covers the whole UTC day.」
- [/guides/segment-filters](https://docs.useplunk.com/guides/segment-filters)：「For custom date fields (`data.<key>`), `within` and `olderThan` only work when the value is stored as an ISO 8601 string... Unix timestamps and other formats won't compare correctly.」；`unit` 取值「`"days"`, `"hours"`, or `"minutes"`」
- [/concepts/segments](https://docs.useplunk.com/concepts/segments)：「Deleting a segment that's referenced by an active campaign (in `DRAFT`, `SCHEDULED`, or `SENDING` state) returns `409 Conflict`. Cancel the campaign or pick a different audience first.」；「Both endpoints **only** accept static segments. Calling them on a dynamic segment returns `400`.」
- [/concepts/segments](https://docs.useplunk.com/concepts/segments)：「The `<slug>` is derived from the segment name: lowercased, accents and punctuation stripped, whitespace replaced with hyphens... `VIP Customers` becomes `segment.vip-customers.entry`. Pick segment names that produce stable slugs — renaming a segment changes the event name.」；「`POST /segments/:id/refresh` — force a count refresh (cheap, no events). `POST /segments/:id/compute` — force a full membership recomputation, which fires any pending entry/exit events.」

## I 解释

Plunk 的分群与过滤只有一套 DSL，却在三个位置上复用：动态 segment 的 `condition`、campaign 的 `audienceCondition`（`audienceType: FILTERED`）、workflow 的 `CONDITION` 步骤。所以真正的成本不是"学会写一次"，而是"写错一次会在三处同时错"。

DSL 的形状是 `{ logic, groups[].filters[] }`，嵌套靠 group 上再挂一个同形状的 `conditions`。关键约束是：组内的多条 filter 之间**恒为 AND**，组与组之间才由该层的 `logic` 连接。这意味着"或"语义不能靠往同一个数组里塞两条 filter 表达，必须新开一个 group 挂 `conditions`。想写 "A AND (B OR C)" 就得用嵌套，不能拍平。

字段侧靠前缀区分查询对象：`email` / `subscribed` / `createdAt` / `data.<path>` / `event.<事件名>` / `email.<互动>` / `segment.<segmentId>`。这里最容易混的是 `segment.` 前缀有两套含义：过滤字段里的 `segment.<segmentId>` 表示"引用另一个 segment 的成员身份"（用 cuid），而事件名里的 `segment.<slug>` 是从段名派生的 slug。两者不可互换。

操作符侧要按字段类型走分支，并记住三分参数规则：大部分比较类需要 `value`，相对时间类（`within` / `olderThan` / `triggeredWithin` / `triggeredOlderThan` / `notTriggeredWithin`）还需要 `unit`（只有 days/hours/minutes），而存在性与成员类（`exists` / `notExists` / `triggered` / `notTriggered` / `memberOfSegment` / `notMemberOfSegment`）两者都不要。漏传 `value` 或 `unit` 是最常见的构造错误。

动态与静态的选型判据是"成员资格由谁决定"：条件随时间自动成立/失效的用动态（必填 `condition`），由人或外部系统点名的用静态（必须省略 `condition`，改用 `/members` 增删）。两者混用会各自静默失败——静态段传 `condition` 被忽略，动态段调 `/members` 返回 400。

最后一条是名字即契约。segment 名派生出 slug，slug 就是 entry/exit 事件名的一部分；改名等于改事件名，所有监听它的 workflow 会静默失活。这与 workflow 触发器事件名"首次执行后不可改"是同一类约束：这两类名字在建立时就该按"长期不变"来取。

时效性也需要单独理解。成员数 `memberCount` 是缓存值，由后台任务按固定节奏刷新，可能比真实状态滞后几分钟，所以它只能当近似值。需要精确值时用 `refresh`（只刷计数，不发事件）；需要立刻把 entry/exit 推给 workflow 时用 `compute`（全量重算并补发 pending 事件）。把这两个端点的区别记成"要不要副作用"就不会用错。

**操作符 × 字段类型速查**

| 字段类型 | 可用操作符 | 是否要 `unit` |
| --- | --- | --- |
| 文本（`email`、字符串型 `data.*`） | `equals` / `notEquals` / `contains` / `notContains` | 否 |
| 布尔（`subscribed`、布尔型 `data.*`） | `equals` / `notEquals` | 否 |
| 日期（`createdAt` / `updatedAt`、ISO 8601 型 `data.*`） | `equals` / `notEquals` / `greaterThan` / `lessThan` / `greaterThanOrEqual` / `lessThanOrEqual` / `within` / `olderThan` | `within` / `olderThan` 需要 |
| 数字 | `equals` / `notEquals` / `greaterThan` / `lessThan` / `greaterThanOrEqual` / `lessThanOrEqual` | 否 |
| 任意 `data.<key>`（存在性） | `exists`（存在且非 null）/ `notExists`（缺失或显式 null） | 否 |
| `event.<name>` / `email.<activity>` | `triggered` / `notTriggered` / `triggeredWithin` / `triggeredOlderThan` / `notTriggeredWithin` | 三个 Within/OlderThan 需要 |
| `segment.<segmentId>` | `memberOfSegment` / `notMemberOfSegment` | 否 |

`unit` 只有三个取值：`days` / `hours` / `minutes`。

**选型速查**

| 维度 | DYNAMIC | STATIC |
| --- | --- | --- |
| 成员资格 | 由 `condition` 实时计算 | 手工策展 |
| `condition` | 必填 | 必须省略（传了被静默忽略） |
| API 增删成员 | 不允许（调 `/members` 返回 400） | `POST` / `DELETE /segments/:id/members` |
| entry/exit 事件 | 仅在 `trackMembership` 开启时发出 | API 增删不发事件 |
| 改过滤器 | 成员数重算 | `condition` 被静默忽略 |
| 典型用途 | 行为化定向 | beta 名单、参会者、外部导入的一次性名单 |
| API 省略 `type` | 默认为 DYNAMIC（于是 `condition` 变必填） | — |

## A1 示例

**示例 1：Engaged users on the Pro plan**（来自 /guides/segment-filters 的 worked example）

- 输入问题："Subscribed Pro-plan users who opened or clicked any email in the last 14 days but haven't clicked anything in the last 30 days."
- 怎么执行：三个 group 用顶层 `logic: AND` 连接——group1 = `subscribed equals true` + `data.plan equals pro`（组内 AND）；group2 挂 `conditions: {logic: "OR", groups: [...]}` 嵌套出 `email.opened triggeredWithin 14 days` / `email.clicked triggeredWithin 14 days`；group3 = `email.clicked notTriggeredWithin 30 days`
- 产出：文档只给 JSON，不给匹配结果（未说明）

**示例 2：New trial users without a purchase**（来自 /guides/segment-filters 的 worked example）

- 输入问题："Subscribed contacts created in the last 7 days who haven't yet triggered a `purchase` event."
- 怎么执行：单个 group 三条 AND——`{subscribed equals true}` + `{createdAt within 7 days}` + `{event.purchase notTriggered}`
- 产出：未说明。要点是内置时间字段配 `within` + `unit: days`，而自定义事件配 `notTriggered` 既不要 `value` 也不要 `unit`

**示例 3：Power users above a tier**（来自 /guides/segment-filters 的 worked example）

- 输入问题："Members of the `power-users` segment whose lifetime value is at least 500."
- 怎么执行：单个 group 两条 AND——`{segment.<powerUsersSegmentId> memberOfSegment}` + `{data.lifetimeValue greaterThanOrEqual 500}`
- 产出：未说明。要点是跨段引用用 `segment.<segmentId>`（cuid，不是 slug），且数字字段不需要 `unit`

**示例 4：静态段批量加人**（来自 /concepts/segments 的 worked example）

- 输入：`POST /segments/:id/members`，body `{ emails: string[], createMissing?: boolean, subscribed?: boolean }`
- 怎么执行：`createMissing: true` 会创建不存在的联系人（除非显式传 `subscribed: false`，否则以已订阅状态创建）
- 产出：`{ added, created, notFound }`；这两个端点只接受静态段（对动态段返回 400），且 API 改成员**不触发** entry/exit 事件

## A2 何时会用

**用户场景**

1. 要建一个"近 14 天打开过邮件但 30 天没点击的 Pro 用户"这类行为化受众，需要把业务语言翻译成条件树 JSON。
2. 分群人数看起来不对（偏少、或一直是 0），要判断是过滤器写错、日期格式不对，还是缓存计数滞后。
3. 想让"进入某个分群"自动触发欢迎邮件，需要开启 `trackMembership` 并决定用 `refresh` 还是 `compute`。
4. 清理不再使用的 segment 时撞上 409，要找出是哪个 campaign 仍在引用它。
5. 准备重命名一个已有的 segment，需要先评估改名会打断哪些 workflow 触发器。

**语言信号**：分群 / segment / dynamic segment / static segment / 过滤器 / filter / 受众 / audience / `audienceType: FILTERED` / condition DSL / `logic` / `groups` / `filters` / `unit` / `memberOfSegment` / `triggeredWithin` / member count / `memberCount` / refresh / compute / segment slug / entry event / exit event

**与相邻能力的区分**

- 本卡负责：分群的选型（动态/静态）、条件树 DSL 的构造、操作符与命名空间、成员计数的刷新口径、slug 与事件名的稳定性。
- `plunk-contacts-and-fields` 负责：`data.*` 字段怎么存、类型怎么推断——本卡只在"过滤器要引用 `data.<path>`"这一点上依赖它。
- `plunk-campaigns-analytics` 负责：把 segment 设为 campaign 受众后的发送与统计口径——本卡只负责 `audienceCondition` 这段 JSON 本身。
- `plunk-workflows` 负责：`CONDITION` 步骤在流程中的位置与分支走向——本卡只负责该步骤里用的同一套过滤条件树。
- `plunk-tracking-events` 负责：`event.<name>` 与 `email.<activity>` 这些事件是怎么被打上来的——本卡只负责如何对它们做过滤。

## E 执行步骤

**输入契约**（必需信息，缺失时先问）

| 必需 | 说明 | 缺失时 |
| --- | --- | --- |
| 业务语言的受众描述 | 要圈定"谁" | 先问；不要凭空替用户造条件 |
| 动态还是静态 | 成员由条件决定还是由人点名 | 先问；判据见 I 段选型速查 |
| 涉及的字段及其类型 | 决定 operator 走哪个分支 | 先问；不确定就先跑 `GET /contacts/fields` 看推断类型 |
| 日期字段的存储格式 | 决定是否能用 `within` / `olderThan` | 先问；非完整 ISO 8601 一律先修数据 |
| 是否要驱动 workflow | 决定是否开 `trackMembership`、用 compute 还是 refresh | 先问 |

**输出契约**

- 一份可直接提交的 `condition` / `audienceCondition` JSON（或 `POST /segments` 的完整请求体）
- 动态/静态选型的结论与理由
- 若涉及事件驱动：`trackMembership: true` 的开关设置 + `refresh` 或 `compute` 的选用结论
- 验收方式：创建对象后用 `GET /segments/:id` 回读 `condition` 与 `memberCount`；需要精确值时先 `POST /segments/:id/refresh`

**步骤**

1. **判定动态 vs 静态**：成员会不会随时间自动变化？会 → DYNAMIC（必须写 `condition`）；由人或外部系统点名 → STATIC（必须省略 `condition`）。完成标准：选型结论有明确理由，且后续步骤与之不矛盾。
2. **把业务语言拆成原子条件**：每条写成 `{field, operator, value?, unit?}`。完成标准：每个原子条件都能对应到 I 段命名空间表里的一个前缀。
3. **按字段类型选 operator**：文本 → `equals`/`notEquals`/`contains`/`notContains`；布尔 → `equals`/`notEquals`；日期 → 比较类 + `within`/`olderThan`；数字 → 比较类（**不要** `unit`）；存在性 → `exists`/`notExists`；事件与邮件互动 → `triggered` 族；跨段 → `memberOfSegment`/`notMemberOfSegment`。完成标准：没有 operator 跨类型使用（例如数字字段上用了 `contains`）。
4. **补参数三分检查**：需要 `value` 的一一核对；相对时间类再补 `unit`（仅 `days`/`hours`/`minutes`）；`exists`/`notExists`/`triggered`/`notTriggered`/`memberOfSegment`/`notMemberOfSegment` 什么都不加。完成标准：没有一个 filter 漏传 `value` 或 `unit`，也没有多传。
5. **处理"或"语义**：任何 OR 都新开一个 group 并挂 `conditions`，不要把多条 filter 塞进同一个 `filters` 数组（组内恒为 AND）。完成标准：形如 "A AND (B OR C)" 的表达式在 JSON 里可见地嵌套，而不是被拍平。
6. **校对日期口径**：`data.<key>` 上的日期必须是完整 ISO 8601 字符串，否则 `within` / `olderThan` 悄悄不匹配（不报错）。若 value 只传 `YYYY-MM-DD`，`equals` 会覆盖整个 UTC 日（`[T00:00:00Z, T23:59:59Z]`），不是时刻相等。完成标准：所有日期 filter 要么用完整 ISO 8601，要么明确知道自己要的是"整日覆盖"。若日期字段当前存的是 Unix 时间戳或 `05/06/2026` 之类，先转 `plunk-contacts-and-fields` 修数据再回来。
7. **校对大小写口径**：`equals` 在 `email` 上大小写不敏感，在 `data.*` 上是精确匹配（大小写敏感）；`contains` 大小写不敏感。完成标准：对 `data.*` 做等于判断的字段已自行规整大小写（否则先转 `plunk-contacts-and-fields` 处理写入侧）。
8. **命名与 slug 决策**：给 segment 取一个你不会改的名字，并手算 slug（小写化、去变音符号与标点、空白转连字符、连续连字符合并）。完成标准：slug 确定且已告知"改名即改事件名"的代价。若确实要改名，转第 11 步。
9. **提交并回读**：`POST /segments`（省略 `type` 即为 DYNAMIC，随之 `condition` 变必填；漏传会 400，`name` 缺失也 400）。完成标准：拿到 201，回读 `condition` 与提交内容一致。若拿到 400，注意该端点只返回 `{ error: "<string>" }` 而非标准错误信封。
10. **处理计数与事件时效**：先把 `memberCount` 当近似值（后台按节奏重算，可能滞后几分钟）。只要准确数字 → `POST /segments/:id/refresh`（廉价、不发事件）；要立刻驱动 workflow 的 entry/exit → `POST /segments/:id/compute`（全量重算并补发 pending 事件）。完成标准：二选一的结论与需求匹配，且已用对应端点取得结果。被别的段频繁嵌套引用的段，开启 `trackMembership` 让成员被直接查表而非每次重算。
11. **删除或重命名的前置检查**：删除被 `DRAFT`/`SCHEDULED`/`SENDING` 状态 campaign 引用的 segment 会返回 409（**注意 DRAFT 也算活跃**）——先取消活动或换受众再删。重命名则先列出所有监听 `segment.<slug>.entry` / `.exit` 的 workflow，改名的同时同步改触发器。完成标准：409 已解除或引用方清单已全部处理。

## B 边界与失效

1. **不要用 `memberCount` 当实时真相**：它是后台任务刷新的缓存值，可能滞后几分钟。要在自动化里做阈值判断（如"人数 > 0 才发"）时，先 `refresh` 再判断。
2. **不要把 `refresh` 当"触发事件"用**：`refresh` 只刷新计数、不发 entry/exit 事件；要补发事件必须用 `compute`。反过来，为了拿一个数字而调 `compute` 是不必要的全量重算。
3. **不要混用动态与静态的 API**：对动态段调 `/segments/:id/members` 返回 400；对静态段在 update 时传 `condition` 会被**静默忽略**（不报错，条件不生效）。这是两类静默/硬失败的不对称。
4. **不要漏传 `value` 或 `unit`**：相对时间类操作符（`within` / `olderThan` / `triggeredWithin` / `triggeredOlderThan` / `notTriggeredWithin`）两者都要；漏传是最常见的构造错误，且失败表现是条件不命中而非报错。
5. **⚠️ 日期口径有硬约束**：`data.<key>` 上的 `within` / `olderThan` 只在值为完整 ISO 8601 字符串时正确，Unix 时间戳与其它格式"不会正确比较"（不报错，只是不匹配）。`equals` 传纯日期时按整个 UTC 日比较，本地时区边界需自行换算——文档未说明跨夏令时的处理，需自行验证。
6. **⚠️ 大小写口径不统一**：`equals` 在 `email` 上不敏感、在 `data.*` 上敏感。从 CRM 同步过来的大小写混乱字段会让 `equals` 静默漏人。
7. **不要给数字字段加 `unit`**：数字比较类不需要 `unit`，加了属于构造冗余；反之日期/事件类漏加则条件失效。
8. **静态段的 API 增删不发事件**：不要指望用 `POST /segments/:id/members` 驱动 entry workflow——entry/exit 只对开启动态追踪（DYNAMIC + `trackMembership`）的段生效。
9. **嵌套未追踪的动态段有性能代价**：多层互相引用的未追踪动态段会放大求值成本。被别人引用的段应开启 `trackMembership`，让其成员被直接查表。
10. **`segment.` 前缀三义**：过滤字段里的 `segment.<segmentId>`（cuid，配 `memberOfSegment`）、事件名里的 `segment.<slug>`（由段名派生）、以及 `data` 里会被静默过滤的 `unsubscribeUrl` 等 URL 变量——三者不要混用。
11. **⚠️ 与删除相关的文档口径**：删除被活跃 campaign 引用的 segment 返回 409，其中"活跃"明确包含 `DRAFT`。这条是文档明说的；但"被 workflow 引用时删除"的后果文档未给出，需自行验证——已知的是删除自定义字段会让引用它的条件静默永不命中，segment 侧是否同构文档未说明。
12. **能力边界**：本卡覆盖过滤条件树与分群生命周期；`audienceType: FILTERED` 在 campaign 上的其余字段、workflow 的 `CONDITION` 步骤在流程中的编排位置、以及 `/workflows/fields` 的探测，分别属于 `plunk-campaigns-analytics` 与 `plunk-workflows`，本卡不展开。

**高危不可撤销操作**

- 删除 segment：若它正被 `DRAFT`/`SCHEDULED`/`SENDING` 的 campaign 引用会被 409 拦下；一旦解除引用后删除成功，文档未给出恢复手段，所有以 `segment.<slug>.entry` / `.exit` 为触发器的 workflow 将失去事件源。
- 重命名 segment：等价于改变事件名，会让监听旧 slug 的自动化静默停止。这是"看起来可撤销、实际上已断裂"的操作，必须先同步所有引用方。
- `POST /segments/:id/compute`：全量重算并补发 pending 的 entry/exit 事件，可能批量触发下游 workflow 与 webhook，属于有外部副作用的动作。
