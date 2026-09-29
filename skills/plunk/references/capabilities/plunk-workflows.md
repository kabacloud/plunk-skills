# 工作流编排

## R 原文依据

- [/concepts/workflows](https://docs.useplunk.com/concepts/workflows)："Workflows are created with `enabled: false`. No executions start until you flip `enabled` to `true` — this is the most common reason 'my workflow isn't firing.'"
- [/concepts/workflows](https://docs.useplunk.com/concepts/workflows)："When you create a workflow via the API, the trigger defaults to `EVENT` with the event name you provide. To use `MANUAL` or `SCHEDULE`, change `triggerType` and `triggerConfig` with `PATCH /workflows/:id` after creation."
- [/concepts/workflows](https://docs.useplunk.com/concepts/workflows)："A workflow's trigger event name **cannot be changed after the first execution** — pick it carefully. Other trigger configuration (delays, conditions inside steps) remains editable."
- [/concepts/workflows](https://docs.useplunk.com/concepts/workflows)：步骤必填配置表——"`SEND_EMAIL`：`templateId`，可选 `from` 覆盖；`DELAY`：`amount`、`unit`（minutes/hours/days）；`WAIT_FOR_EVENT`：`eventName`、`timeout`（秒）；`CONDITION`：一个过滤表达式（与 segment filters 同形）；`WEBHOOK`：`url`，可选 `method`/`headers`/`body`；`UPDATE_CONTACT`：`data` 对象；`EXIT`：可选 `reason`。"
- [/concepts/workflows](https://docs.useplunk.com/concepts/workflows)："For `CONDITION` steps, each transition carries a `branch` discriminator (`\"yes\"` or `\"no\"`)... When deleting a step in the middle of a chain, pass `?splice=true` on `DELETE /workflows/:id/steps/:stepId`... otherwise the deletion leaves the graph disconnected."
- [/concepts/workflows](https://docs.useplunk.com/concepts/workflows)："A workflow is **locked** for structural edits while it has active (`RUNNING` or `WAITING`) executions... Disable the workflow and wait for active executions to complete, or `POST /workflows/:id/executions/cancel-all`. Cosmetic changes (renaming, editing template content referenced by steps) don't require this."
- [/concepts/workflows](https://docs.useplunk.com/concepts/workflows)："Each execution carries a `context` JSON object that's merged with the contact's `data` when rendering templates and evaluating conditions... you can pass an initial `context` — this is how you parameterise per-execution variables (a coupon code, a referrer name) without storing them on the contact."
- [/api-reference/overview](https://docs.useplunk.com/api-reference/overview)："`POST /workflows` — Create a workflow. Always starts with `triggerType: EVENT` and `enabled: false`."；"`DELETE /workflows/:id` — Delete a workflow. Active executions must be cancelled or completed first."

## I 解释

Workflow 是 Plunk 里唯一的"持续自动化"载体：一份 workflow 记录 + 一张由 step 与 transition 构成的图 + 每个进入的联系人一份 execution。与 campaign 的差别就是"每个联系人各跑一遍、一直跑"，所以它的设计重心不在内容而在**图结构与生命周期**。

入口被设计成单点：系统自动建一个 `TRIGGER` 步骤，它是唯一入口，其余全靠加节点加边拼出来。`triggerType` 只有三种，但 API 创建时恒为 `EVENT` 且 `enabled: false`——这是一对默认值陷阱：你要的一切非默认行为（MANUAL、SCHEDULE、真正开始触发）都得创建后再 `PATCH`。更狠的是触发事件名在首次执行后锁死，所以事件命名是一次性决策，必须用带命名空间的名字。

图的连通性由两个参数显式维护：`CONDITION` 的两条出边必须带 `branch: yes/no`（否则引擎不知道走哪边），删中间节点必须带 `?splice=true`（否则默认留一个断图，且断图不会报错、只是静默断流）。这两个参数是"API 手工建图"最容易漏的地方。

生命周期侧有三组开关值得单独记：`enabled`（默认关，不触发的头号原因）、`allowReentry`（默认关，决定同一联系人能否多次进入）、以及"活跃执行锁定"（有 RUNNING/WAITING 时禁止结构性改动，但改名、改被引用模板内容这类外观改动随时可做）。最后一条其实是给了你一条安全通道：**改文案不必停 workflow，改结构必须清空在飞执行**。

execution 的 `context` 解决的是"变量归属"问题：属于这个人的写 `data.*`，属于这一次的走 MANUAL 启动时的 `context`，属于单次发送的走 `data` 的 `{ value, persistent: false }`。`context` 在渲染模板与求值 CONDITION 时与 `data` 合并，所以模板里写法一致，但不会污染联系人档案。

## A1 示例

**示例 1 · Welcome series：DELAY 三段序列**（来自 `/concepts/workflows`）

- 输入/问题：注册后第 0/2/7 天各发一封，只走一次。
- 怎么执行：`EVENT` trigger on `signed_up` → SEND_EMAIL（welcome）→ DELAY `amount: 2, unit: days` → SEND_EMAIL（tips）→ DELAY `amount: 5, unit: days` → SEND_EMAIL（upgrade nudge）；`allowReentry` 保持 `false`，最后 `PATCH` 打开 `enabled`。
- 产出/预期输出：文档未说明（原书未给出执行结果）。

**示例 2 · Webhook fan-out：一次触发三种下游**（来自 `/concepts/workflows`）

- 输入/问题：购买完成后要同时写 CRM、给联系人打标、发收据。
- 怎么执行：`EVENT` trigger on `purchase.completed` → `WEBHOOK` 步骤打到 CRM → `UPDATE_CONTACT` 写 `{ tier: "customer" }` → `SEND_EMAIL` 发收据。
- 产出/预期输出：文档未说明（原书未给出执行结果）。

**示例 3 · Re-engagement：segment 退出 + 等待开启分支**（来自 `/concepts/workflows`）

- 输入/问题：用户从活跃分群掉出后做回流，按是否在 7 天内开启邮件分支。
- 怎么执行：trigger 用 `segment.active-users.exit` → SEND_EMAIL（we miss you）→ `WAIT_FOR_EVENT`（`eventName: email.opened`, `timeout` 秒）→ `CONDITION`（是否回流），两条出边分别标 `branch: "yes"` / `"no"` → 各自后续动作。
- 产出/预期输出：文档未说明（原书未给出执行结果）。

**示例 4 · 用 UPDATE_CONTACT 把"事件事实"物化为联系人状态**（来自 `/recipes/waitlist`）

- 输入/问题：报名后要能按"等待列表用户"分群，而不是去过滤"是否触发过 waitlist.joined"。
- 怎么执行：在 `SEND_EMAIL` 前插一个 `UPDATE_CONTACT` 步骤，写入 `{ "stage": "waitlist", "waitlistSource": "{{event.referrer}}" }`；之后按 `stage == "waitlist"` 建 segment。文档原话："This is cleaner than filtering on 'ever fired waitlist.joined'."
- 产出/预期输出：文档未说明（原书未给出执行结果）；可判定的是联系人上出现 `stage` 字段，且事件上下文 `{{event.referrer}}` 能在 UPDATE_CONTACT 里直接引用。

## A2 何时会用

**用户场景**

1. 要做一个"注册后第 0/2/7 天各发一封"的多段序列，不知道延迟与分支怎么摆。
2. 通过 API 建完 workflow，事件明明在 Activity 里，但一条执行都没有。
3. 线上 workflow 正在跑，要加/删一个步骤、换触发事件或换触发类型，改动被拒；或要删链条中间的一个步骤。
4. 要按收件地址/事件内容分流（support / sales / billing，或 spam verdict 判定）。
5. 要给每次执行带一次性变量（优惠码、推荐人），又不想写进联系人档案。

**语言信号**

`工作流` / `workflow` / `自动化` / `automation` / `journey` / `欢迎邮件序列` / `welcome series` / `不触发` / `workflow isn't firing` / `enabled` / `allowReentry` / `trigger event` / `DELAY` / `WAIT_FOR_EVENT` / `CONDITION` / `branch` / `splice` / `execution` / `FAILED` / `cancel-all` / `编辑被锁` / `workflow locked` / `execution context`

**与相邻能力的区分**

- 本卡负责 workflow 的图结构、触发器、步骤类型、执行状态机与编辑锁定。
- `plunk-tracking-events` 负责"事件怎么上报、哪些事件名被保留"——本卡只消费事件，不定义事件。
- `plunk-templates-liquid` 负责 SEND_EMAIL 引用的模板内容与变量渲染——本卡只管 `templateId` 这个引用。
- `plunk-webhooks` 负责 WEBHOOK 步骤的出站契约（payload、超时、重试）——本卡只管把它当作图中的一个节点。
- `plunk-segments-filters` 负责 CONDITION 表达式的 DSL 细节——本卡只要求"表达式与 segment filter 同形"。
- `plunk-campaigns-analytics` 负责一次性群发，不进入本卡的持续编排范畴。

## E 执行步骤

**输入契约（必需信息）**

1. 触发方式：`EVENT`（给事件名）/ `MANUAL` / `SCHEDULE`（给 `triggerConfig`）。
2. 每一步的节奏与动作：是否要 DELAY（amount+unit）、是否要 WAIT_FOR_EVENT（eventName+timeout 秒）、分支条件、末端动作（SEND_EMAIL 的 `templateId` / WEBHOOK 的 url / UPDATE_CONTACT 的 data / EXIT）。
3. 生命周期开关：`enabled` 何时打开、`allowReentry` 取值。
4. 缺失时先问：① 触发事件名（首次执行后不可改，必须现在定）；② 这是"只走一次"还是"可重复进入"；③ 目标 workflow 当前有没有 RUNNING/WAITING 的执行。

**步骤**

1. **定触发**：确定 `triggerType` 与事件名。

   | `triggerType` | 何时触发 | 需要的 config |
   | --- | --- | --- |
   | `EVENT` | 匹配事件被 track 到时（自定义事件、系统事件如 `email.received` / `contact.subscribed`、segment entry/exit） | `eventName` |
   | `MANUAL` | 仅当显式调用 `POST /workflows/:id/executions` | 无（可传 `context`） |
   | `SCHEDULE` | 由 `triggerConfig`（如 cron 表达式）定义的周期调度 | `triggerConfig` |

   事件名用带命名空间的形式（如 `waitlist.joined`），不要用以后可能想复用的泛化名（`signup`）。完成标准：事件名已定稿并记入事件字典——因为它首次执行后不可改，改只能重建 workflow。
2. **建 workflow**：`POST /workflows`。完成标准：拿到 workflow id；同时明确知道它现在是 `triggerType: EVENT` + `enabled: false`，不会有执行启动。
3. **若需要 MANUAL / SCHEDULE**：`PATCH /workflows/:id` 改 `triggerType` 与 `triggerConfig`。完成标准：`GET /workflows/:id` 返回的 `triggerType` 已是目标值。若第 1 步定的就是 `EVENT`，跳过本步。
4. **建步骤**：`POST /workflows/:id/steps` 逐个添加。类型与必填配置是一一对应的：

   | Step type | 作用 | 必填 config |
   | --- | --- | --- |
   | `TRIGGER` | 自动创建的入口，持有触发配置 | `eventName`（`EVENT` 触发器）；不用手工建 |
   | `SEND_EMAIL` | 用模板给联系人发邮件，变量取自 `data` + context | `templateId`，可选 `from` 覆盖 |
   | `DELAY` | 固定时长暂停 | `amount` + `unit`（`minutes` / `hours` / `days`） |
   | `WAIT_FOR_EVENT` | 等到指定事件被 track 到，带超时兜底 | `eventName` + `timeout`（**秒**） |
   | `CONDITION` | 按联系人或事件数据分支，两条出边 | 过滤表达式（与 segment filter 同形） |
   | `WEBHOOK` | 向外部 HTTPS 端点发 JSON | `url`，可选 `method` / `headers` / `body` |
   | `UPDATE_CONTACT` | patch 联系人 `data`（如打标） | `data` 对象 |
   | `EXIT` | 终止 execution | 可选 `reason`（记为 `exitReason`） |

   完成标准：所有步骤已创建且必填项齐全（漏 `unit`、或把 `timeout` 当分钟写，是最常见的两类错误）。
5. **连边**：`POST /workflows/:id/transitions`。凡是从 `CONDITION` 出发的边，**必须**带 `branch: "yes"` 或 `"no"`。完成标准：每个 CONDITION 步骤恰好两条出边，且 branch 取值合法。
6. **按需物化状态**：如果后续还要按"是否经历过这个流程"分群，在发信前加 `UPDATE_CONTACT` 写入标记字段（如 `{ stage: "waitlist" }`），再按该字段建 segment。完成标准：联系人上出现该字段；不要去过滤"曾触发过某事件"。
7. **处理一次性变量**：变量属于"这一次"而不是"这个人"时，走 MANUAL 执行 `POST /workflows/:id/executions` 传入 `context`，模板与 CONDITION 里按合并后的作用域引用。完成标准：变量可用且联系人 `data` 里没有它。
8. **打开开关**：`PATCH /workflows/:id` 设 `enabled: true`；按"首次触达型关 / 可重复参与型开"设 `allowReentry`。完成标准：目标事件被 track 后，`GET /workflows/:id/executions` 出现新执行。
9. **若改动被拒（结构锁）**：先判断改动性质——增删步骤、改触发事件、换 triggerType 属结构性改动，需要先解锁；改名、编辑步骤引用的模板内容属外观改动，直接改即可。结构性改动的解锁二选一：禁用 workflow 并等活跃执行跑完，或 `POST /workflows/:id/executions/cancel-all`。完成标准：无 RUNNING/WAITING 执行后改动成功提交。
10. **删中间步骤**：`DELETE /workflows/:id/steps/:stepId?splice=true`。完成标准：删除后图仍连通（不带 `splice=true` 会留下断图，且不报错、只静默断流）。
11. **删整个 workflow**：先 `POST /workflows/:id/executions/cancel-all` 或等活跃执行跑完，再 `DELETE /workflows/:id`。完成标准：删除成功；未清空执行时删除会被拒。
12. **若不触发，按固定顺序诊断**：① `enabled` 是否为 true（头号原因）→ ② `triggerType`/`triggerConfig` 是否正确（API 创建总是 EVENT + disabled）→ ③ 触发事件名是否精确匹配 → ④ `allowReentry` 是否为 false 而该联系人已经进过一次 → ⑤ 事件是否真的产生了（看 Activity 里有没有对应事件）→ ⑥ CONDITION 是否把事件全过滤掉了。完成标准：定位到六处之一；前四处是配置问题，第五、六处要回到上游或条件表达式。
13. **读执行状态**：execution 的六个状态决定你该查什么——

   | Status | 含义 | 该做什么 |
   | --- | --- | --- |
   | `RUNNING` | 正在处理某一步（或即将处理） | 正常，等 |
   | `WAITING` | 卡在 `DELAY` 或 `WAIT_FOR_EVENT` | 正常，等；同时说明 workflow 处于结构锁状态 |
   | `COMPLETED` | 成功跑完图 | 无需处理 |
   | `EXITED` | 撞到 `EXIT` 步骤 | 看 `exitReason` 判断是否符合预期 |
   | `FAILED` | 不可恢复错误 | 见下 |
   | `CANCELLED` | 经 API 或 dashboard 手动取消 | 确认是谁取消的 |

14. **若执行落在 FAILED**：按 `/concepts/workflows` 的两个典型成因查——模板找不到（SEND_EMAIL 的 `templateId` 失效）、或 webhook 步骤返回非 2xx。完成标准：修掉根因后重新触发；⚠️ 文档口径不一致，两者并列，不要替文档做取舍——本页把 FAILED 描述为 "webhook returned non-2xx after retries"，而 `/guides/webhooks` 明确写 "No automatic retries"，工程上按"无自动重试"做防御。
15. **⚠️ 走到 API schema 边界时换路径**：`POST /workflows`、`/steps`、`/transitions` 的完整请求 schema 文档未给出，已知约束只有"创建即 EVENT + `enabled: false`"、步骤类型枚举、以及 `branch` 与 `splice` 两个参数。若需要建一个超出这些已知约束的图：优先用 dashboard（或 MCP 工具）建图，或用 `GET /workflows/fields` 探测 CONDITION 可用字段集合。完成标准：明确告诉用户"这一步只能走 dashboard"，不要凭印象编造字段名。

**输出契约**

- 产出：一份 workflow 定义（trigger + steps + transitions）+ 已确认的 `enabled` / `allowReentry` 取值 + 一次成功的触发验证。
- 验收：① `GET /workflows/:id` 返回的步骤与边和图设计一致，CONDITION 的两条出边 branch 齐全；② 用真实事件打一次，`GET /workflows/:id/executions` 出现执行且状态不是 FAILED；③ 需要 MANUAL/SCHEDULE 时 `triggerType` 已切换到位；④ 若只交付到 API 已知边界，明确列出哪些步骤配置需要用户在 dashboard 完成。
- 交付后必须交代的三件事（否则用户会在下一轮踩坑）：① 触发事件名已锁死，改名只能重建；② 改动结构的两种解锁路径及各自代价；③ 该 workflow 是否依赖了文档未给出 schema 的字段。

## B 边界与失效

1. **⚠️ Workflow API 只有路由表、没有完整 schema（`p153`）**：除 `branch` 与 `splice` 外，`POST /workflows`、`POST /workflows/:id/steps`、`POST /workflows/:id/transitions` 的字段细节文档未给出。不要编造字段名与请求体，改用 dashboard / MCP 建图，或用 `GET /workflows/fields` 探测 CONDITION 字段集。
2. **⚠️ webhook 重试语义两处矛盾（`p154`）**：`/guides/webhooks` 说"No automatic retries"，`/concepts/workflows` 的 FAILED 又说"webhook returned non-2xx after retries"。两者并列，不要替文档做取舍；工程上按"无自动重试"设计。
3. **不要用本卡做一次性群发**：给一批人群在同一时刻发一封，是 `plunk-campaigns-analytics` 的职责；workflow 是"每个联系人各跑一遍"的持续结构，用它做群发会为每个联系人各建一份执行。
4. **不要在活跃执行期做结构性改动**：有 RUNNING/WAITING 执行时增删步骤、改触发事件、换 triggerType 会被拒。别为了"快点上线"直接 cancel-all——那会丢掉在飞联系人的后续步骤；先评估是禁用等待还是全部取消。
5. **不要指望改回触发事件名**：首次执行后即锁死。重建 workflow 会割裂历史执行，且旧 workflow 的活跃执行要先清空才能删。
6. **高危不可撤销操作**：`POST /workflows/:id/executions/cancel-all`（在飞执行全部取消，不可恢复）、`DELETE /workflows/:id`（workflow 与图结构删除）、`DELETE /workflows/:id/steps/:stepId` 不带 `splice=true`（静默留下断图）、删 `WAIT_FOR_EVENT` 超时兜底（可能让执行无限等待）。其中 cancel-all 与 workflow 删除的 recoverability 为 0，执行前必须确认活跃执行可以丢弃。
7. **不要把手写的一次性变量塞进 `contact.data`**：这会把"属于这一次"的数据永久留在联系人档案上，污染后续分群与渲染。走 execution `context` 或 `{ value, persistent: false }`。
8. **CONDITION 表达式细节不在本卡**：字段与操作符的完整 DSL 见 `plunk-segments-filters`；本卡只保证"两条出边 + branch"这个结构正确。
9. **不要用 workflow 表达"在某时刻给所有人发一封"**：SCHEDULE trigger 的语义文档只描述为"由 `triggerConfig`（如 cron）定义的周期调度"，未说明它对哪些联系人生效，落地前需自行验证。
10. **不要用 API 探测未文档化的字段来"补全"建图**：这属于超出能力边界的猜测，一旦字段不存在只会得到 400，且无法判断是字段错还是格式错。
11. **不要在不知道活跃执行代价的情况下 cancel-all**：在飞执行被取消后不会自动恢复，已经走到一半的联系人会失去后续步骤；若这批联系人很重要，禁用 + 等待是更安全的选择。
12. **不要用 workflow 承担事务性一致性**：执行是异步、逐联系人、可能落在 FAILED 的，且失败不会自动重投（见第 2 条）。需要强一致的下游写入，应在接收侧做对账而不是假设 workflow 一定跑完。
13. **不要假设 SCHEDULE 会覆盖"所有联系人"**：文档只说它由 `triggerConfig`（如 cron）定义周期调度，未说明调度时对哪些联系人生效；也不要用它替代 campaign 的一次性群发。
