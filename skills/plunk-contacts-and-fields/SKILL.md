---
name: plunk-contacts-and-fields
description: |
  当你要决定业务属性该放 contact 顶层还是 data.*、某个字段为何在分群 UI 里类型不对、
  传了空值却发现旧值没被清掉、要按 5MB 上限批量导入名单并轮询 job、或准备删除一个
  可能仍被 segment/campaign/workflow 引用的自定义字段时使用。
  提供字段与期望类型，产出 schema 划分、特殊值写入语义、类型污染复位流程、
  CSV 导入对账口径与"先 usage 后 delete"的清理顺序。
  不负责过滤条件怎么写（见 plunk-segments-filters），也不负责 subscribed 的投递语义（见 plunk-subscription-consent）。
  triggers: 自定义字段, custom fields, "data.*", 类型推断, type inference, 保留键, 空值, null, "persistent: false", CSV 导入, job 轮询, 字段删除, coverage
metadata:
  cangjie.generated-by: cangjie-tools v2.5.0
  cangjie.capability-id: cap.plunk-docs.contacts-and-fields
  cangjie.capability-revision: 1
  cangjie.bundle-id: bundle.plunk-docs
  cangjie.source-title: Plunk Developer Docs
  cangjie.tags: contacts, custom-fields, data-model, import
---
# 联系人建模与自定义字段

## R 原文依据

- [/concepts/contacts](https://docs.useplunk.com/concepts/contacts)：「Contact data types are inferred from the **first non-null value** Plunk sees for a given key in your project... once a key is typed as `number`, subsequent string values for that key are still stored, but the **inferred type** stays `number` for the segment filter UI.」
- [/concepts/contacts](https://docs.useplunk.com/concepts/contacts)：特殊值处理表 —— 「Empty string (`""`) — Ignored - field is not stored or updated. `null` — Delete - field is removed from contact data. Other values — Stored/updated normally.」；非持久化值「Send a value as `{ value, persistent: false }` to use it once for template rendering without storing it」
- [/concepts/contacts](https://docs.useplunk.com/concepts/contacts)：「Some keys are reserved by Plunk and managed at the contact level (not inside `data`). You can read them but you can't set them through `data`: `email`, `createdAt`, `updatedAt`, `subscribed`, `snoozedUntil`」；静默过滤键「`id`, `plunk_id`, `plunk_email`, `unsubscribeUrl`, `subscribeUrl`, `manageUrl`」
- [/guides/custom-fields](https://docs.useplunk.com/guides/custom-fields)：「`data` is patched — only the keys you send change. Other keys keep their existing values.」；「To use date-aware operators (`within`, `olderThan`) on a custom field, store the value as a full ISO 8601 string — `"2026-05-06T12:00:00Z"` works, `"05/06/2026"` doesn't.」
- [/guides/custom-fields](https://docs.useplunk.com/guides/custom-fields)：「`GET /contacts/fields/:field/usage` — lists every segment, campaign, and workflow that references the field. Run this before deleting... Deleting only affects contact data — segments and workflows that referenced the field continue to exist but their conditions on that field will stop matching anything.」
- [/guides/importing-contacts](https://docs.useplunk.com/guides/importing-contacts)：「**File size**: up to 5 MB per upload... **Reserved column names**: `id`, `subscribed`, `createdAt`, `updatedAt`, and the auto-generated URL variables are silently filtered out... Leading-zero values (`01234`), `+`-prefixed numbers, and scientific notation stay strings so IDs, zip codes, and phone numbers aren't corrupted.」
- [/api-reference/contacts/createContact](https://docs.useplunk.com/api-reference/contacts/createContact)：「`200` — Contact updated. `201` — Contact created. `_meta`: `isNew`, `isUpdate`.」；同一端点的 `400` 只返回 `{ "error": "string" }`，`401` 才是标准信封。

## I 解释

Plunk 的数据模型只有一个一等公民：contact（一个 email 收件人）。contact 顶层只有五个字段（`email` / `subscribed` / `createdAt` / `updatedAt` / `snoozedUntil`），其余一切业务属性都必须挂在 `data` 对象下。这条边界不是风格问题——顶层字段由 Plunk 自己管理，你往 `data` 里塞 `email` 或 `subscribed` 不会生效，甚至会被静默丢弃。

`data` 的写入语义必须先记牢三件事：它是 patch（只改你传的键）、空字符串表示"忽略"而不是"清空"、真正的删除只能靠显式传 `null`。这套语义在 `POST /contacts`、`PATCH /contacts/:id`、`/v1/track` 的 data、`/v1/send` 的 data 上一致，所以只要在一个地方理解错，四条入口会一起错。

第二件要理解的是类型推断的"采样 + 粘滞"机制。类型按项目内第一次见到的非 null 值采样一次就固定，之后就算你写进完全不同类型的值，存储照存，但推断类型不变——而这个推断类型只影响 segment 过滤 UI 渲染什么控件。后果是"值存下来了、但过滤器口径不对"这类静默不一致。唯一的复位手段是删字段再按正确类型重导，因此字段类型本质上是一次性决策。

第三件是治理代价。`data.*` 同时喂三个消费方：模板变量、segment 过滤器、workflow 的 CONDITION 分支。加一个字段是三处影响面，删一个字段则让三处引用静默变成"永不命中"而不报错。所以字段生命周期的两端（命名、删除）都必须当作契约来管理：命名阶段就假定它永不改名，删除阶段必须先查 usage 再动手。

第四件是入口差异会咬人。同一个"订阅"语义在四条入口上默认值不同：`/v1/track` 打到不存在的邮箱会自动建联系人且默认 subscribed；`POST /contacts` 默认 false；`/v1/send` 新联系人默认 false；CSV 导入默认 subscribed。所以"我从哪个入口第一次见到这个邮箱"直接决定了它一开始是否可发营销邮件——批量导入历史名单时这条差异就是合规风险的来源。

第五件是异步作业的形态。超过千条量级就该走 CSV 导入：上传拿 `jobId`、轮询到终态、按 `imported / updated / skipped / errors` 对账、修好错误行重传。重传之所以安全，是因为匹配键是 email 且合并方式是 patch，天然幂等——这让"改一列再传一遍"成为批量更正的正规手段，而不是权宜之计。

**语义速查**

| 维度 | 规则 |
| --- | --- |
| 谁是一等公民 | contact（一个 email 收件人），其它对象都挂在它或它的 `data` 上 |
| contact 顶层字段 | `email` / `subscribed` / `createdAt` / `updatedAt` / `snoozedUntil`（可读，不可通过 `data` 设） |
| 业务属性放哪 | `data.<path>`，支持嵌套（`data.profile.tier`），但 dashboard 较难发现，高基数字段建议平铺 |
| `data` 写入 | patch：只改传入的键 |
| `""` | 忽略，不存不覆盖 |
| `null` | 删除该键 |
| `{ value, persistent: false }` | 仅本次渲染，随后丢弃 |
| 类型推断 | 首个非 null 值 → number → boolean → date（ISO 8601）→ string（兜底），项目级 sticky |
| 推断类型影响面 | 只影响 segment 过滤 UI 的控件；存储的值原样保存 |
| 类型复位 | `DELETE /contacts/fields/:field` 后按目标类型重新导入（全项目生效，不可逆） |
| 批量入口 | `POST /contacts/import`（≤5MB、UTF-8、首列 email）→ 轮询 `GET /contacts/import/:jobId` |
| upsert 回执 | 新建 201 / 更新 200，响应带 `_meta.isNew` / `_meta.isUpdate` |
| 删除联系人 | `DELETE /contacts/:id` → 204 无响应体，永久不可逆 |

## A1 示例

**示例 1：一次 track 写入四种类型，随后在三处消费**（来自 /guides/custom-fields 的 worked example）

- 输入：`POST /v1/track` 带 `data: {firstName:"Ada", plan:"pro", signupDate:"2026-05-06T12:00:00Z", lifetimeValue:240}`
- 执行：字段被推断为 string / string / date / number；随后在模板 `{{firstName ?? "there"}}` 与 `{% if plan == 'pro' and lifetimeValue > 500 %}`、segment `{field:"data.plan",operator:"equals",value:"pro"}`、segment `{field:"data.signupDate",operator:"within",value:30,unit:"days"}` 三处引用
- 产出：`signupDate` 因存的是完整 ISO 8601 字符串，`within` 可用；若改成 `"05/06/2026"` 则日期操作符失效（文档未给出匹配结果的数值）

**示例 2：一次性重置码用非持久化值**（来自 /concepts/contacts 的 worked example）

- 输入：
  ```json
  { "email": "user@example.com",
    "data": { "firstName": "Ada",
              "resetCode": { "value": "ABC123", "persistent": false } } }
  ```
- 执行：`firstName` 落到联系人上长期保存，`resetCode` 只参与本次模板渲染
- 产出：文档说明 `resetCode` 在本次发送后被丢弃；避免了把一次性凭据留在 dashboard 与任何 API 持有者可见的 `data` 里

**示例 3：CSV 批量导入与 job 轮询**（来自 /guides/importing-contacts 的 worked example）

- 输入：`email,firstName,plan,signupDate` 三行（Ada/pro、Grace/enterprise、Linus/free），`curl -X POST /contacts/import -F "file=@contacts.csv"` ≤5MB
- 执行：取得 `{jobId:"imp_abc123", status:"queued"}` → 每隔几秒轮询 `GET /contacts/import/imp_abc123` 直到 `completed` / `failed`
- 产出：样例回执 `totalRows 12000 / imported 11985 / updated 8 / skipped 0`，`errors` 中 row 47、row 1042 因 `invalid_email` 被拒；修好错误行重传是幂等的（按 email 合并，不产生重复）

**示例 4：字段审计与清理**（来自 /guides/custom-fields 的 worked example）

- 输入：`GET /contacts/fields` 返回 `[{field:"email",type:"string",isCustom:false,coverage:1.0},{field:"data.plan",type:"string",isCustom:true,coverage:0.62},{field:"data.signupDate",type:"date",isCustom:true,coverage:0.95}]`
- 执行：用 coverage 判陈旧 → 清理前 `GET /contacts/fields/:field/usage` 查引用 → 逐个改条件/改受众/取消活动 → 再 `DELETE /contacts/fields/:field`
- 产出：文档说明删除只影响联系人数据，引用方仍存在但对这个字段的条件不再匹配任何联系人

## A2 何时会用

**用户场景**

1. 从外部系统（CRM / 自建用户表）同步用户属性，要决定哪些字段进 `data.*`、日期怎么存才能让分群的 `within` 生效。
2. 分群 UI 里某个字段的控件类型不对（日期字段不给日期选择器、数字字段不能比大小），怀疑类型被污染。
3. 想清空或重置某个联系人的某个属性，发现传 `""` 没有效果。
4. 要批量导入几万条名单（含从未 opt-in 的用户），需要控制导入的默认订阅行为并看懂 job 回执。
5. 准备下线一个自定义字段，需要评估它会弄坏哪些 segment / campaign / workflow。
6. 一封邮件里要带密码重置码、magic link、一次性 token，不能让它留在联系人档案上。

**语言信号**：自定义字段 / custom fields / contact data / `data.*` / 类型推断 / type inference / 保留键 / reserved keys / 空值清空 / `null` vs `""` / 非持久化 / `persistent: false` / CSV 导入 / import contacts / 5MB / job 轮询 / 字段删除 / field usage / coverage

**与相邻能力的区分**

- 本卡负责：contact 上"放什么、怎么存、怎么删、怎么批量导入"——即数据建模与字段治理。
- `plunk-segments-filters` 负责：拿到这些字段后怎么写过滤条件树、操作符与 `unit` 怎么配。
- `plunk-subscription-consent` 负责：`subscribed` / snooze / 退订与合规语义——本卡只在"它是 contact 级保留键、不能通过 `data` 设置"这一点上涉及。
- `plunk-templates-liquid` 负责：`{{firstName}}`、`??` 回落、Liquid 语法——本卡只负责字段值本身是否存在。
- `plunk-api-essentials` 负责：密钥、幂等、错误信封与分页——本卡只引用 `POST /contacts` 的 201/200 语义。

## E 执行步骤

**输入契约**（必需信息，缺失时先问）

| 必需 | 说明 | 缺失时 |
| --- | --- | --- |
| 项目与 secret key | 所有 contact 写接口都要 `sk_` | 先问；不要用 `pk_` |
| 目标 scope | 单个联系人 / 批量 CSV / 全项目字段治理 | 先问，三者流程不同 |
| 字段名与期望类型 | 新增或修复字段时必须明确 | 先问；日期一律按完整 ISO 8601 |
| 待删字段是否被引用 | 删除类操作的前置 | 必须自己跑 usage，不能凭印象 |

**输出契约**

- 单个写入：一次 `POST /contacts` 或 `PATCH /contacts/:id` 请求体 + 状态码判定（201 新建 / 200 更新）
- 批量导入：`jobId` + 轮询到终态后的 `imported / updated / skipped / errors` 对账结果
- 字段治理：`GET /contacts/fields` 的字段清单（含 type / coverage）+ usage 引用方清单 + 清理后删除动作
- 验收方式：回读 `GET /contacts/:id` 或 `GET /contacts/fields` 确认类型与值；删除类操作验收引用方条件已改或已下线

**步骤**

1. **定 schema**：列出要存的业务属性，逐个判定——是否属于 contact 顶层五字段之一？是则走请求专用字段；否则进 `data.*`。完成标准：没有 `email` / `subscribed` / `createdAt` / `updatedAt` / `snoozedUntil` 出现在 `data` 里。
2. **过保留键黑名单**：检查字段名是否撞上静默过滤键。⚠️ 文档口径不一致，两者并列，不要替文档做取舍：`/concepts/contacts` 列 6 个（`id`、`plunk_id`、`plunk_email`、`unsubscribeUrl`、`subscribeUrl`、`manageUrl`）；`/guides/custom-fields` 与 `/v1/send` 列 7 个（同样 6 个 + `email`）；CSV 导入的"保留列名"是第三套（`id`、`subscribed`、`createdAt`、`updatedAt` + 三个自动生成的 URL 变量）。完成标准：按三套的并集做防御，字段名一个都不撞。
3. **定类型**：给每个字段确定目标类型并先写一条样本值。日期必须是完整 ISO 8601（`2026-05-06T12:00:00Z`），否则 segment 的 `within` / `olderThan` 不可用。完成标准：`GET /contacts/fields` 里该字段的 `type` 与预期一致。
4. **写入**：`data` 只传要改的键（patch 语义）。要删除某键传 `null`，**不要**传 `""`（空串被忽略，旧值原样保留）。一次性值用 `{ value, persistent: false }`。完成标准：回读联系人，`data` 中目标键已变更、其余键未动。
5. **读回执**：`POST /contacts` 按 email upsert——`201` 为新建、`200` 为更新，响应 `_meta.isNew` / `_meta.isUpdate` 可据此决定是否发欢迎流程。完成标准：客户端按状态码分支，而不是假定永远 200。若拿到 `400`，注意该端点只返回 `{ "error": "<string>" }`（非标准信封），解析时需同时兼容两种形状。
6. **批量导入走异步作业**：>1000 条或已有 CSV 就走 `POST /contacts/import`。前置检查：≤5MB、UTF-8、有表头且首列是 `email`、日期列 ISO 8601、布尔列 `true/false/yes/no`（大小写不敏感）。超过 5MB 自行拆文件。完成标准：拿到 `jobId` 且首次轮询返回 `queued`/`processing`。
7. **轮询到终态**：每隔几秒 `GET /contacts/import/:jobId` 直到 `completed` 或 `failed`（文档未给出推荐间隔与超时上限，自行设定上限避免无限轮询）。完成标准：`imported + updated + skipped + errors.length` 与 `totalRows` 对得上（恒等关系文档未明示，属按字段语义推断）。
8. **纠错重传**：按 `errors[].row` / `reason` 修 CSV，重跑同一文件——按 email 合并更新，天然幂等，不产生重复。完成标准：重跑后 `errors` 为空或仅剩预期行。
9. **处理导入的默认订阅**：导入的联系人默认 `subscribed`（与 `POST /contacts` 默认 false 相反）。名单未经 opt-in 时用 `POST /contacts/bulk-unsubscribe` 保留档案但不再发信，或用 workflow 过滤。完成标准：非 opt-in 名单的 `subscribed` 已按合规要求处理。
10. **字段治理（删除前必查）**：跑 `GET /contacts/fields` 看 coverage 找陈旧字段 → 跑 `GET /contacts/fields/:field/usage` 列出所有引用方 → 逐个改条件 / 改受众 / 取消 campaign → 最后 `DELETE /contacts/fields/:field`。完成标准：usage 返回为空，或每个引用方都已显式处理。
11. **类型污染修复**：若某字段推断类型不对（混入多类型），顺序是 `DELETE /contacts/fields/:field` → 用正确类型全量重新导入。注意删除是全项目生效且不可逆。完成标准：`GET /contacts/fields` 中该字段 `type` 已变为目标类型。若静默过滤键撞名导致值写不进去，转第 2 步改名。

## B 边界与失效

1. **不要用 `""` 清空字段**：空串被忽略，既不存也不覆盖。要删除必须传 `null`。这是最高频的静默失败——调用返回成功，旧值原封不动，分群与模板继续命中旧值。
2. **不要通过 `data` 改 `email` / `subscribed`**：它们是 contact 级保留键，塞进 `data` 无效。改 `email` 走 `PATCH /contacts/:id` 的专用字段，且若项目内已有别的联系人用该邮箱会返回 409。
3. **⚠️ 静默过滤键清单三处不一致**（见 E 段第 2 步），工程上按并集防御；撞名时不存储也不报错，排查成本极高。
4. **删除自定义字段是广域且不可逆的动作**：它会从项目内**每个**联系人上移除该键，且引用它的 segment / campaign / workflow 不会报错，只是条件从此不再匹配任何人（分群成员清空、workflow 分支永远走 else）。这是"静默失效"而非"报错失败"，往往很晚才被发现。
5. **删除联系人是永久性的**：`DELETE /contacts/:id` 返回 204 无响应体（客户端不要尝试解析 JSON），不可恢复。需要"软删除"语义请用 `subscribed: false` 或 snooze 替代。
6. **类型推断复位有代价**：`DELETE /contacts/fields/:field` 是唯一稳妥的重置方式，但它同时清掉所有联系人上的该字段值。因此类型决策要前置，不要指望事后无痛苦修正。
7. **不要把秘密放进 `data`**：`data` 在 dashboard 与任何持有 API 权限者处可见。重置码、magic link、一次性 token 一律走 `{ value, persistent: false }`。
8. **CSV 的数字推断有例外**：前导零（`01234`）、带 `+`、科学计数法会被刻意保持为字符串，以保护 ID / 邮编 / 电话。若这些值还要参与 `greaterThan` / `lessThan` 比较，得在写入侧先规整格式——文档未说明规整后的推荐写法。
9. **导入默认订阅存在合规风险**：CSV 导入的联系人默认 subscribed，可能让从未 opt-in 的人进列表。不要跳过第 9 步。
10. **客户端解析 400 要做容错**：`POST /contacts` / `POST /segments` / `POST /templates` 的 400 只返回 `{ error: "<string>" }`，与 errors 页声明的"统一错误信封"矛盾；同一端点的 401 又是标准信封。不能假定所有错误都带 `{success, error{...}}`。
11. **能力边界**：字段级 `GET /contacts/fields/:field/values`（distinct 值）与 dashboard 手动添加联系人属于文档提及但本卡不展开的路径；项目级 reset 不可逆，不在本卡内给出恢复做法。
12. **字段改名等同于断裂**：文档明确警告"Pick stable field names. Renaming a field means updating every segment, template, and workflow that references it."。改名没有迁移工具，只能手工同步三处引用。
13. **嵌套路径可用但不好发现**：`data.profile.tier` 在模板与过滤器里都能工作，但在 segment UI 里较难被找到；面向运营自助操作的字段建议用平铺键。
14. **CSV 布尔口径与概念页略有出入**：导入页说明 `true/false/yes/no`（大小写不敏感）都会存为 boolean，而 contacts 概念页的 boolean 只列 `true`/`false`。⚠️ 文档口径不一致，两者并列，不要替文档做取舍；实操上以导入页的四值为准即可覆盖概念页的子集。
15. **不要指望"改 email" 是低成本操作**：它既是 contact 级保留键（不能通过 `data` 改），又在撞上项目内已有联系人时返回 409。业务上需要换邮箱时，先确认目标邮箱未被占用。
16. **列联系人时别被三个默认值误导**：`limit` 超过 100 会被静默 clamp 到 100；`subscribed` 省略表示**已订阅与已退订都返回**（不是默认只返回已订阅）；`total` 只在首页（未传 cursor 时）有值，翻页时返回 0。写全量遍历脚本时必须按 cursor 翻页，且不能用后续页的 `total` 做进度条。
17. **删除前没有批量清理工具**：`GET /contacts/fields/:field/usage` 只给引用方清单，文档未提供"引用计数"或批量解除引用的端点，只能手工逐个处理（改条件 / 改受众 / 取消 campaign）。规模大时要预留人工成本。

**高危不可撤销操作**

- `DELETE /contacts/fields/:field`：从项目内**所有**联系人上移除该字段，且让引用方的条件静默永不命中。执行前必须完成 E 段第 10 步的 usage 检查。
- `DELETE /contacts/:id`：永久删除联系人，204 无响应体，无恢复路径。
- 类型污染的复位（`DELETE /contacts/fields/:field` + 重导）：同样清零全项目该字段的值，属于"用一次数据损失换类型正确"的取舍，执行前确认有全量重导的数据源。
- `POST /contacts/bulk-delete`（每批最多 1000 个 id）：批量永久删除，文档未给出撤销手段。
