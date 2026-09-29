# 营销活动与统计口径

## R 原文依据

- [/concepts/campaigns](https://docs.useplunk.com/concepts/campaigns)："Unlike workflows, a campaign sends once at a single point in time (immediately or at a scheduled time) and is then frozen."
- [/concepts/campaigns](https://docs.useplunk.com/concepts/campaigns)："DRAFT — Created but not yet scheduled or sent. Editable... SENT — Send is complete. Stats are final. CANCELLED — Terminal state from DRAFT, SCHEDULED, or SENDING... Once a campaign is SENT or CANCELLED it can't be edited or resent — duplicate it instead."
- [/concepts/campaigns](https://docs.useplunk.com/concepts/campaigns)："audienceType decides which companion field is required — SEGMENT requires segmentId, FILTERED requires audienceCondition, and ALL requires neither."（同源另一处："`audienceType` — Who this campaign is sent to. One of ALL, SEGMENT, or FILTERED."）
- [/concepts/campaigns](https://docs.useplunk.com/concepts/campaigns)："Counts are recomputed from the underlying emails and events on every call, so they are authoritative rather than eventually consistent. Every rate is a percentage of `sentCount`."
- [/concepts/campaigns](https://docs.useplunk.com/concepts/campaigns)："Bounces and complaints flip the contact to unsubscribed as well, but they are reported in their own column and never in `unsubscribedCount` — the same lost recipient counted twice would overstate both."
- [/concepts/campaigns](https://docs.useplunk.com/concepts/campaigns)："Complaint rate deserves attention out of proportion to its size: mailbox providers measure it in fractions of a percent, and a sustained rate above roughly 0.1% puts sending at risk... which is why the dashboard leads that card with the count rather than the percentage."
- [/concepts/campaigns](https://docs.useplunk.com/concepts/campaigns)："Cancellation stops further sending but cannot recall emails that have already been handed off to the recipient's mail server."
- [/concepts/campaigns](https://docs.useplunk.com/concepts/campaigns)："Send a single test email with `POST /campaigns/:id/test`. Body: `{ email: \"you@example.com\" }` — a single address, not an array... useful for previewing personalisation against your own contact data."

## I 解释

Campaign 是"一次性的批量群发"：在某个时间点（立即或 `scheduledFor` 指定）发一次，然后冻结。它与 workflow 的唯一结构性差别就是"只发一次"，因此所有设计都围绕"这次发送"这个单位，而不是持续运行的受众。

第一层是生命周期与不可变点。状态机是 DRAFT（可编辑）→ SCHEDULED（可取消）→ SENDING（队列在处理收件人，可取消）→ SENT（完成，统计定稿）；CANCELLED 是可从 DRAFT/SCHEDULED/SENDING 进入的终态。两个不可逆点必须提前说清：**SENT 或 CANCELLED 之后既不能编辑也不能重发**，要重来只能 `POST /campaigns/:id/duplicate` 复制出一份新的 DRAFT；**cancel 只能拦住尚未交给对方邮件服务器的部分**，已交接的邮件收不回。所以"发完发现错字"这件事没有补救路径，只有"复制—修改—再发"和"尽早 cancel 止损"。

第二层是受众。`audienceType` 是必填字段，它决定哪个伴生字段必填：`ALL` 两个都不要（项目内全部已订阅联系人）、`SEGMENT` 必须给 `segmentId`、`FILTERED` 必须给 `audienceCondition`（与 segment 的 condition 同 JSON 形状，适合一次性受众不想存成可复用 segment）。缺对应字段返回 400。这里最容易混的是两个字段：`type` 是**内容类型**（MARKETING/TRANSACTIONAL/HEADLESS，默认 MARKETING，决定页脚与是否跳过退订者），`audienceType` 是**受众类型**，API 文档专门提醒不要混淆。同理，`status` 这类业务枚举传错值直接 400，而 `sort` / `dir` 传错会静默回退到默认值——客户端封装时不能假设"非法参数都会报错"。

第三层是统计口径，这是本卡最容易被误读的部分。读 stats 要同时记住三件事：① 三个"损失"列**互斥**——`bouncedCount` / `complainedCount` / `unsubscribedCount` 对同一收件人只计一次，硬退与投诉虽然也把联系人翻成退订，但只进各自的列，**绝不进** `unsubscribedCount`（否则同一人被计两次、两边都虚高）；② `unsubscribedCount` 只算能通过退订链接上 `?e=` 参数归因到本活动的主动退订，dashboard 开关、CSV 导入、`PATCH /contacts/:id`、以及该参数存在之前发出的邮件里的链接都无法归因，因此不计入任何 campaign；③ 所有比率的分母一律是 `sentCount`，且计数每次调用都从底层邮件与事件重算，是权威值而不是最终一致值——不必等，但每次读都有重算成本。

第四层是行动阈值与展示原则。投诉率要按绝对数而非百分比看：服务商按千分之几计量，**持续高于约 0.1% 就会危及发信能力**，单次活动哪怕只有几起投诉也是值得处理的信号；dashboard 那张卡因此用 count 领头（百分比小到看不出变化）。同时必须打折看互动数据：打开/点击追踪存在系统性假阳性（禁图导致无记录、企业安全网关预取造成假打开、Apple Mail Privacy Protection 掩盖真实打开、安全工具预扫链接造成假点击），所以"未打开"不能等同于"不感兴趣"，高价值判定优先用点击或自定义事件。

把这些口径合在一起，就是读一次活动数据的固定顺序：

| 读什么 | 口径 | 容易读错的地方 |
| --- | --- | --- |
| `bouncedCount` / `complainedCount` / `unsubscribedCount` | 三列互斥，同一收件人只计一次 | 不能相加；硬退与投诉虽把人翻成退订，但**不进** unsubscribedCount |
| 所有 rate | 分母一律是 `sentCount` | 不是 recipients，也不是 delivered |
| 计数新鲜度 | 每次调用从底层邮件与事件重算，权威而非最终一致 | 不必等，但每次读都有重算成本 |
| 投诉 | 看绝对数，约 0.1% 是持续值警戒线 | 百分比小到看不出变化，"几起"也是信号 |
| 打开/点击 | 带系统性假阳性 | "未打开"不等于"不感兴趣" |

还有两个"为什么"值得记住。比率一律以 `sentCount` 为分母，是因为 campaign 关心的是"成功交出去的那些邮件里发生了什么"，用 recipients 作分母会让被跳过（退订、无效）的人稀释指标；计数每次调用都重算而不是最终一致，是为了让 stats 可以直接当权威值用——不必等流水线收敛，但相应地每次读都有重算成本。

还有一个结构上的判断要先做：campaign 与 workflow 的区别只有一条——**是否只发一次**。要持续触发、要等用户行为、要分支，就属于 workflow，不要用一个 campaign 加一堆手工补发去模拟。

## A1 示例

**示例 1 · 三种受众类型的字段组合**（来自 `/concepts/campaigns` 的 anatomy 表）

- 输入/问题：要发一封新品公告，受众说法在"全量"、"某个 segment"、"临时拼一批人"之间摇摆。
- 怎么执行：全量 → `audienceType: "ALL"`，不传 segmentId/audienceCondition；复用已有分群逻辑 → `"SEGMENT"` + `segmentId`；一次性受众、不想存成可复用 segment → `"FILTERED"` + `audienceCondition`（与 segment 过滤器同 JSON 形状）。
- 产出/预期输出：请求体字段与 audienceType 一一对应。缺伴生字段返回 400（"segmentId missing for a SEGMENT campaign, or audienceCondition missing for a FILTERED campaign"）。

**示例 2 · 发完发现错字**（来自 counter-example e54 与 e55）

- 输入/问题：活动已 SENT，正文有个错字；团队想直接 PUT 更新或再调一次 `/send`。
- 怎么执行：更新与重发都不可用（SENT/CANCELLED 后不可编辑或重发）。若活动还在 SCHEDULED/SENDING，先 `POST /campaigns/:id/cancel` 止损（只能停未发出的部分）；若已 SENT，用 `POST /campaigns/:id/duplicate` 复制出新的 DRAFT 再改再发。
- 产出/预期输出：一份新的 DRAFT 副本；已交给对方邮件服务器的那部分邮件仍然会被收件人收到——文档明确无法召回。

**示例 3 · 测试发送核对个性化**（来自 `/concepts/campaigns` 的 worked example）

- 输入/问题：想在正式群发前确认变量渲染正确。
- 怎么执行：`POST /campaigns/:id/test`，body 是 `{ email: "you@example.com" }`——**单个地址，不是数组**。测试发送走与真实发送相同的模板变量解析，但总是发到你指定的地址。
- 产出/预期输出：一封到自己邮箱的样例邮件。⚠️ 文档口径不一致，两者并列，不要替文档做取舍：concepts 页只说"单个地址"，MCP 页补充"测试收件人必须是项目成员，其他地址会被拒"——若测试被拒，先确认收件地址是否属于该项目成员。

**示例 4 · 读一次活动统计**（合成演练，按 `/concepts/campaigns` 的口径可复算）

- 输入/问题：`GET /campaigns/:id/stats` 返回 sentCount 10000、bouncedCount 120、complainedCount 9、unsubscribedCount 40，团队按"退订 169 人"汇报。
- 怎么执行：三列互斥，不能相加——退订列只有 40（可归因的主动退订），硬退 120 与投诉 9 各在自己的列；投诉率 = 9 / 10000 = 0.09%，逼近 0.1% 警戒线且是绝对数 9 起，应作为信号处理；打开率按 sentCount 为分母计算，且读数时已知会带系统性假阳性。
- 产出/预期输出：分类汇报（硬退 120 / 投诉 9 / 主动退订 40）+ 投诉优先的动作项（查内容与频控、确认退订链接可用）。sentCount 为 0 时的除法行为文档未说明。

**示例 5 · 未验证域名建活动被 403**（来自 counter-example e51）

- 输入/问题：项目刚上线，先建好活动再补域名验证；或 `from` 用了同项目下另一个还没验证的域。
- 怎么执行：创建与更新都会校验 `from` 的域名，未注册或未完成 DNS 验证返回 **403**（不是 400/422）。正确顺序是先完成域名验证（DKIM 3 CNAME + SPF TXT + bounce MX），再创建/更新带 `from` 的资源。
- 产出/预期输出：不调整顺序的话请求直接 403，且常被误判成权限问题而白查一轮 API key。

**示例 6 · 复制上一次的群发**（来自 `/concepts/campaigns` 的 duplicating 节；本例为按文档规则可复算的合成演练，HVAC 之类周期性促销同理）

- 输入/问题：上一封季度促销已 SENT，这一季度要再发一封内容几乎相同的。
- 怎么执行：`POST /campaigns/:id/duplicate` 会生成一份内容、受众、设置都相同的**新 DRAFT**，在其上改文案后重走测试发送与发送。
- 产出/预期输出：一份新的 DRAFT campaign（文档未给出响应字段清单）。这也是 A/B 变体与周期性发送的标准做法——不要试图编辑或重发已 SENT 的那一份。

## A2 何时会用

**用户场景**

1. 要创建一次 newsletter / 公告 / 促销群发，需要确定受众类型与必填字段。
2. 活动已发出，需要判断是否该 cancel、能不能改内容或重发。
3. 看 dashboard 或 stats 时不确定"退订数"到底包含哪些人、比率的分母是什么。
4. 投诉率或退信率看起来很小，想知道要不要处理、阈值在哪。
5. 想用打开/点击数据做再营销或对外报 KPI，需要知道数据的可信度边界。
6. 要复制一次"上次那封群发"（A/B 变体或周期性发送），或发送前先试发一封。

**语言信号（中英双写）**

`营销活动` / `campaign` / `群发` / `broadcast` / `newsletter` / `DRAFT` / `SCHEDULED` / `SENDING` / `SENT` / `CANCELLED` / `audienceType` / `segmentId` / `audienceCondition` / `campaign stats` / `sentCount` / `bouncedCount` / `complainedCount` / `unsubscribedCount` / `complaint rate` / `0.1%` / `duplicate` / `test send`

**与相邻能力的区分**

- 本卡负责：一次性群发的全生命周期（创建 → 排期/发送 → 取消/复制）与这次发送的统计口径。
- `plunk-workflows` 负责持续运行的自动化旅程（trigger、step、execution）；判断标准是"只发一次 vs 持续运行"。
- `plunk-segments-filters` 负责 segment 与过滤条件 DSL 本身；本卡只决定"用哪种受众类型、伴生字段填什么"。
- `plunk-templates-liquid` 负责模板内容与 `type`（内容类型）语义；本卡只用它的结论（MARKETING/HEADLESS 会跳过退订者）。
- `plunk-sending-health` 负责退信/投诉的日常列表卫生与演练；本卡只负责"这次活动的投诉率怎么读、要不要处理"。
- `plunk-tracking-events` 负责打开/点击追踪的模式与事件；本卡只负责这些数字进入活动统计后该怎么打折看。

**按诉求快速分流**：

- "要发一封一次性群发" → 本卡（创建 → 发送 → 统计）。
- "要持续触发、等用户行为、有分支" → `plunk-workflows`。
- "分群/过滤条件怎么建" → `plunk-segments-filters`。
- "文案与模板渲染" → `plunk-templates-liquid`。
- "退信投诉怎么处理" → `plunk-sending-health`。

## E 执行步骤

**输入契约**

必需信息：
1. 受众：是"全量已订阅"、某个 segment 的 id、还是一次性过滤条件（`audienceCondition` 的 JSON）。
2. 内容类型 `type`（省略即 MARKETING）与发件人 `from`（域名必须已验证）。
3. 发送时机：立即还是 `scheduledFor`（ISO 8601）。
4. 读取统计时：campaign id 与要看的时间点。

缺失时先问：受众能不能复用已有 segment（能复用就别用 FILTERED 重复造一份）、这封是否真的属于"只发一次"（否则应走 workflow）、发信域名是否已验证。

**输出契约**

产出：可直接提交的 `POST /campaigns` 请求体（含 audienceType 与对应伴生字段）、发送/排期/取消的动作序列、以及一份按三列互斥口径解读的 stats 结论（含投诉率是否触及 ~0.1% 的判断）。验收：① 创建返回的新活动状态为 DRAFT 且 `type` 与预期一致；② 发送后状态推进到 SENDING→SENT 且可在 dashboard 看进度；③ stats 的三个损失列能逐项说出归因来源。

0. **先读状态再决定动作**：`GET /campaigns/:id` 看 status——`DRAFT` 可编辑；`SCHEDULED` / `SENDING` 可 cancel；`SENT` / `CANCELLED` 是终态，只能 duplicate。完成标准：动作与状态匹配（终态上不尝试编辑或重发）。若目标是新建活动，跳到第 1 步；若目标是读统计，跳到第 9 步。
1. **先定"是不是 campaign"**：只发一次 → campaign；持续触发/有等待与分支 → 转 `plunk-workflows`。完成标准：能一句话说明为什么不是 workflow。
2. **选受众类型并补齐伴生字段**：`ALL` 不传伴生字段；`SEGMENT` 必传 `segmentId`；`FILTERED` 必传 `audienceCondition`（与 segment 过滤器同形）。完成标准：audienceType 与伴生字段一一对应，不缺也不多；若这次受众以后还要用，改成先建 segment。
3. **确认内容类型与退订语义**：`type` 省略默认 MARKETING。MARKETING/HEADLESS 会自动跳过已退订联系人；TRANSACTIONAL 会投递给所有匹配受众、不看订阅状态——只在真正的事务性内容上使用。完成标准：明确知道这次发送会不会被退订者跳过；不要为"送达率"选类型。
4. **校验发信域名**：`from` 必须落在已验证域名上，创建和**更新**都会校验，未验证返回 403（不是 400/422）。完成标准：先完成域名验证再建资源，"先建后补验证"不可行。
5. **发送前先试发**：`POST /campaigns/:id/test`，body 为 `{ email: "<单个地址>" }`（传数组不合法），用真实联系人数据核对个性化渲染。完成标准：自己收到一封渲染正确的样例。若被拒，按 ⚠️ 文档口径核对收件地址是否为项目成员。
6. **发送或排期**：立即发用 `POST /campaigns/:id/send`；定时用同一个端点传 `scheduledFor`，或创建时带 `scheduledFor`。完成标准：状态从 DRAFT 变为 SENDING（有 scheduledFor 时先为 SCHEDULED）；发送是后台异步的，受众越大耗时越久，进度在 dashboard 的 campaign 详情页看。
7. **需要变体或复投就复制**：`POST /campaigns/:id/duplicate` 会生成一份内容、受众、设置都相同的新 DRAFT。完成标准：得到新的 DRAFT id，改完再走第 5–6 步；不要试图编辑或重发已 SENT/CANCELLED 的活动。
8. **误发止损**：仍在 SCHEDULED/SENDING 时立刻 `POST /campaigns/:id/cancel`。完成标准：明确告知干系人"已交给对方邮件服务器的部分收不回"，并估算已发出范围。
9. **读统计**：`GET /campaigns/:id/stats`，按三列互斥解读——`bouncedCount`（`Email.bouncedAt`，硬退会抑制联系人）/ `complainedCount`（`Email.complainedAt`，会抑制联系人）/ `unsubscribedCount`（靠退订链接 `?e=` 归因）。完成标准：不把三列相加，且能说明未归因退订为什么不在任何 campaign 里。
10. **算比率并先看投诉**：所有 rate = 该指标 count / `sentCount`；投诉率按绝对数优先判断，持续高于约 0.1% 即视为发信风险信号（哪怕只有几起）。完成标准：给出"投诉 N 起 / 率 X%"并给出动作（查内容相关性、确认退订链接可用、控频、查名单质量）。
11. **给互动数据打折**：打开率会受禁图、企业网关预取、Apple MPP 影响而虚高或失真，点击率会受安全工具预扫描影响。完成标准：对外报 KPI 或驱动再营销时，优先用点击 / 自定义事件 / 自定义字段这类主动信号，而不是"未打开"。
    - ⚠️ 文档口径不一致，两者并列，不要替文档做取舍：打开/点击追踪存在**系统性假阳性**（预取与预扫描会产生假打开/假点击，Apple Mail Privacy Protection 掩盖真实打开），文档没有给出剔除方法；需要更可信的互动口径时只能改用更强的信号自行验证。
    - ⚠️ 文档口径不一致，两者并列，不要替文档做取舍：`unsubscribedCount` 的归因口径在 campaigns 页限定为"主动退订且可归因"，而 list-hygiene 页明确 snooze（暂停订阅）也计入 campaign 的退订数。统计时按两种口径分别记录，不要自行合并。
    - ⚠️ 文档口径不一致，两者并列，不要替文档做取舍：campaigns 概念页列出的反范式化统计字段为 `totalRecipients, sentCount, deliveredCount, openedCount, clickedCount, bouncedCount`，而 stats 端点与术语页还说会返回 complained 与 unsubscribed 及各自比率。取数时以实际响应为准。

## B 边界与失效

1. **终态后不可编辑或重发**：SENT / CANCELLED 之后更新被拒、重发未定义，只能 duplicate 出新的 DRAFT。发前的测试发送是唯一低成本防线。
2. **cancel 不能召回**：只能停止尚未发出的部分；已交给对方邮件服务器的邮件一定会被收到。误发止损要尽早，且必须如实告知影响范围。
3. **统计口径的加总是错的**：三个损失列互斥，硬退/投诉虽然把联系人翻成退订，但绝不进 `unsubscribedCount`；把一个收件人计两次会让两边都虚高。
4. **归因有盲区**：dashboard 开关、CSV 导入、`PATCH /contacts/:id`、以及 `?e=` 参数存在之前发出的邮件里的退订链接，都无法归因，因此不计入任何 campaign——不要用 campaign 的退订数去对账全站退订量。
5. **投诉率不能按百分比直觉判断**：约 0.1% 是持续值警戒线，且"几起投诉就是信号"；dashboard 用绝对数领头正是为了让小数字可见。忽略它会直接危及发信能力。
6. **互动数据带系统性假阳性**：预取、预扫描与 Apple MPP 会让打开/点击失真，用它做"未打开就补发"这类自动化会误触发。
7. **不要用 TRANSACTIONAL 绕过退订**：TRANSACTIONAL 会投递给所有匹配受众、无视订阅状态，只在真正的事务性内容上使用；用它发营销内容会直接推高投诉率。
8. **参数容错策略不一致**：`status` 这类业务枚举传错直接 400，`sort` / `dir` 传错静默回退默认值。客户端封装时不能假设"所有非法参数都会报错"。
9. **高危/不可逆操作**：触发 `/send` 后活动进入不可编辑流程；`DELETE /campaigns/:id` 在有 active executions 时返回 409（删除本身不可撤销）；`from` 所在域名的验证状态变更会让创建/更新直接 403。
10. **dashboard-only 边界**：发送进度、统计卡片的展示形态（如投诉卡用 count 领头）只能在 dashboard 观察，API 只给 stats 的数值；本卡的"权威重算"只适用于 `GET /campaigns/:id/stats` 返回的字段，不代表 dashboard 上的所有视图都有等价 API。
