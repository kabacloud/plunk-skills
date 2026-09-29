---
name: plunk-api-essentials
description: |
  当你第一次接 Plunk API 要决定 pk_/sk_ 放哪里、调用返回 401/403/422/429/500 要怎么分支、
  要给发信或打事件加重试又怕重复执行、或要写全量遍历与批量同步脚本时使用。
  提供 project 与端点，产出端点族双套解析规则、幂等键生成与 4xx/5xx/409 三条重试分支、
  状态码↔错误码映射与容量阈值速查。
  不负责单个端点的业务字段语义（见 plunk-transactional-send），也不负责限流的环境变量配置（见 plunk-self-hosting）。
  triggers: API key, pk_, sk_, 401, 幂等, Idempotency-Key, 409, 限流, 429, requestId, 分页, cursor, hasMore, rate limit
metadata:
  cangjie.generated-by: cangjie-tools v2.5.0
  cangjie.capability-id: cap.plunk-docs.api-essentials
  cangjie.capability-revision: 1
  cangjie.bundle-id: bundle.plunk-docs
  cangjie.source-title: Plunk Developer Docs
  cangjie.tags: api, auth, idempotency, errors, rate-limit
---
# API 调用基石：密钥、错误、幂等与限流

## R 原文依据

- [/guides/api-keys](https://docs.useplunk.com/guides/api-keys)："Public (pk_): POST /v1/track only — safe in Client-side code (browser, mobile apps). Secret (sk_): Every other endpoint — sending email, contacts, segments, campaigns, templates, workflows, domains, billing — Server-side only."
- [/guides/api-keys](https://docs.useplunk.com/guides/api-keys)："If the key prefix doesn't match the endpoint (e.g. you use `pk_` on `/v1/send`), the API returns `401` with code `INVALID_API_KEY`."
- [/guides/api-keys](https://docs.useplunk.com/guides/api-keys)："Rotating regenerates **both** keys simultaneously — there is no way to rotate one without invalidating the other... There's no grace period — plan a brief deployment window if you have multiple consumers."
- [/guides/idempotency](https://docs.useplunk.com/guides/idempotency)："2xx success — Kept ... 4xx client error — Released ... 5xx server error — Kept: The request may have partially completed. Refusing the retry is the whole point of the key."
- [/guides/idempotency](https://docs.useplunk.com/guides/idempotency)："Plunk **refuses** a reused key; it does not replay the original response... it does not return the original email or event ID. If you need that ID, record it when the first request succeeds."
- [/api-reference/errors](https://docs.useplunk.com/api-reference/errors)："success — Always `false` for errors"；统一信封为 `error: { code, message, statusCode, requestId, errors, details, suggestion }` + `timestamp`。
- [/api-reference/overview](https://docs.useplunk.com/api-reference/overview)："API requests — 1000 requests/minute per project."（自托管文档另有第二套口径，见 B 段）
- [/api-reference/overview](https://docs.useplunk.com/api-reference/overview)："Pass the response's cursor value as the next request's cursor query parameter. When `hasMore` is false, you've reached the end. The total count is only included on the first page (when no cursor is supplied)."

## I 解释

Plunk 的 API 面可以分成三层理解，任何一次调用失败都只可能落在这三层之一：**你是谁**（密钥）、**这次请求算不算数**（幂等）、**结果怎么读**（响应信封与错误码）。

密钥层的设计动机是"最小暴露面"。一个项目只有一对 key，`pk_` 被人为阉割成只能调 `POST /v1/track`，这样浏览器里埋点不必把整站写权限下发到前端；`sk_` 覆盖其余一切端点，只能待在服务端。项目归属由 key 反推，所以请求里永远不需要传 project ID——反过来说，拿错项目的 key 就会静默写进另一个项目。

幂等层解决的是"重试即重复"的真实副作用。Plunk 的语义是 **at-most-once 而非 exactly-once-with-replay**：重复 key 会被 409 拒绝，但不会把第一次的响应再发一遍。所以键的占用/释放规则（2xx 保留、4xx 释放、5xx 保留）本质是"这次到底写没写"的判据，而"409 拿不回原 ID"要求你在首次成功时就把 ID 落库。

响应层有两个容易踩的非对称：public API（`/v1/send`、`/v1/track`、`/v1/verify`）套 `{success, data}` 信封，dashboard 端点（contacts/templates/campaigns/segments/workflows）直接返回裸资源；以及 HTTP 状态码和 `error.code` 不是一一对应（403 既可能是权限，也可能是项目被禁用或需要验证邮箱）。因此文档反复强调"按 code 分支，不要按状态码分支"。

最容易被当成小事的其实是 requestId：它贯穿 API→数据库→队列→worker，且成功响应也有 `X-Request-ID` 头。没有它，事后排障和支持沟通都无从下手。

## A1 示例

**示例 1 · 三种语言调用 /v1/send，以及 pk_ 误用的 401**（来自 `/guides/api-keys`）

- 输入/问题：后端要发一封密码重置邮件，手上有 `sk_` 与 `pk_` 两把 key。
- 怎么执行：cURL / JavaScript / Python 三份等价调用，均为 `Authorization: Bearer $PLUNK_SECRET_KEY` + `Content-Type: application/json`，body `{to, subject, body}`；JavaScript 版从 `process.env.PLUNK_SECRET_KEY` 取值。若误用 `pk_` 调 `/v1/send`，返回 401 `INVALID_API_KEY`，message 会出现 "This endpoint requires a secret key (sk_*), not a public key"。
- 产出/预期输出：200 + `{success: true, data: {...}}`；误用 pk_ 时为上述 401 信封。

**示例 2 · `receipt-order-1234` 幂等键与 409**（来自 `/guides/idempotency`）

- 输入/问题：订单收据邮件，防止重复提交导致重复发信。
- 怎么执行：`curl -X POST /v1/send -H "Authorization: Bearer ..." -H "Idempotency-Key: receipt-order-1234" -d '{"to":"customer@example.com","subject":"Your receipt","body":"<p>Thanks for your order!</p>"}'`。首次 200，键被占用；24 小时内重放返回 409 `IDEMPOTENCY_KEY_REUSED`，`details` 含 `key` / `originalRequest` / `originalRequestAt` / `originalStatusCode`。
- 产出/预期输出：409 响应只说明"第二次没执行"；`originalStatusCode` 为 `200` 表示首次已完成，为 `null` 表示首次仍在飞行中、本次是并发竞争失败方。原 `emailId` 不会出现在 409 里。

**示例 3 · 5xx 之后键被烧毁**（来自 `/guides/idempotency`）

- 输入/问题：`/v1/send` 首次返回 500，业务侧想"原样重试"。
- 怎么执行：按释放矩阵判断——5xx 属于 Kept，同一 key 重试会直接 409。
- 产出/预期输出：确认请求确实没生效后，换一个新 key 重试；若多收件人且中途 5xx，部分收件人可能已收到，键保持占用正是为了阻止盲重试。

**示例 4 · 游标分页全量遍历**（按 `/api-reference/overview` 规则可复算的合成演练）

- 输入/问题：要把项目里 10,000 个联系人全量导出。
- 怎么执行：首页 `GET /contacts?limit=100`（不传 cursor），读 `total = 10000`、`cursor = "def456"`、`hasMore = true`；之后每页带上一步的 `cursor`，直到 `hasMore` 为 false。
- 产出/预期输出：全量 10,000 条；注意第 2 页起响应里 `total` 为 `0`，这是"未计算"而非"总数为 0"，总数只能用首页那个值。

## A2 何时会用

**用户场景**

1. 第一次接 Plunk，不知道该把哪把 key 放哪里、为什么前端埋点不能用 `sk_`。
2. 调用返回 401 / 403 / 422 / 429 / 500，需要判断该重试、该改请求还是该停工找支持。
3. 要给"发邮件 / 打事件"这类有副作用的操作加重试，担心网络抖动导致重复发信。
4. 要写批量同步脚本，需要知道分页怎么翻、每页上限多少、批量端点一次能吃多少。
5. 密钥进了公开仓库或承包商离职，需要走轮换与定损流程。

**语言信号（中英双写）**

`API key` / `密钥` / `pk_` / `sk_` / `401` / `INVALID_API_KEY` / `轮换` / `rotate keys` / `幂等` / `idempotency` / `Idempotency-Key` / `409` / `重复发送` / `duplicate email` / `限流` / `rate limit` / `429` / `requestId` / `X-Request-ID` / `分页` / `pagination` / `cursor` / `hasMore`

**与相邻能力的区分**

- 本卡负责：跨端点的通用调用契约——认证、幂等、错误信封、分页、容量阈值、限流。
- `plunk-transactional-send` 负责 `/v1/send` 这个具体端点的字段与必填组合。
- `plunk-contacts-and-fields` 负责联系人与 `data.*` 字段本身的语义。
- `plunk-domain-deliverability` 负责 403 背后最常见的根因（发信域名未验证）。
- `plunk-self-hosting` 负责自托管侧的限流开关与环境变量。
- `plunk-billing-limits` 负责 402 两类错误码对应的套餐与额度细节。

## E 执行步骤

**输入契约**

必需信息：
1. 调用哪个端点（决定用 `pk_` 还是 `sk_`，决定有没有 `{success,data}` 信封）。
2. 该端点是否属于 public API 写端点（只有 `POST /v1/send` 与 `POST /v1/track` 支持幂等键）。
3. 这次操作重复一次会不会产生真实副作用（重复邮件、重复事件）。
4. 是否已有 requestId 的落日志机制。

缺失时先问：目标端点路径、运行环境（浏览器/服务端/自托管）、是否已有重试逻辑、是否需要在失败后拿回业务 ID。

**容量阈值速查**

| 项 | 阈值 | 要点 |
| --- | --- | --- |
| `limit`（游标分页） | 默认 20，上限 100 | 超过 100 会被 clamp 到 100 |
| `pageSize`（页码分页） | 上限 100 | `GET /segments/:id/contacts` |
| `POST /contacts/lookup` | ≤500 个邮箱/次 | 批量存在性检查 |
| `POST /contacts/import` | ≤5MB（multipart） | 异步，返回 `jobId` |
| bulk-subscribe / unsubscribe / delete | ≤1000 个联系人 ID/次 | 异步，返回 `jobId` |
| `Idempotency-Key` | 1–255 可打印 ASCII，TTL 24h | 仅 `/v1/send` 与 `/v1/track` 支持 |
| 429 退避节奏 | 1s / 2s / 4s / 8s | 来自 errors 页的排障建议 |
| 域名验证后台检查 | 每 5 分钟一次 | 可 `GET /domains/:id/verify` 强制 |
| 日志保留 | 30 天（见 `plunk-data-retention`） | requestId 取证受此窗口约束 |

**HTTP 状态码 ↔ 错误码速查**（按 code 分支，不要只看左列）

| 状态码 | 错误码 | 含义要点 |
| --- | --- | --- |
| 400 | `BAD_REQUEST`、`INVALID_REQUEST_BODY` | 请求格式/请求体损坏；部分端点非标准信封 |
| 401 | `UNAUTHORIZED`、`INVALID_CREDENTIALS`、`MISSING_AUTH`、`INVALID_API_KEY` | 认证失败或缺 Authorization 头 |
| 402 | `BILLING_LIMIT_EXCEEDED`、`UPGRADE_REQUIRED` | 两类：限额 vs 需升级套餐 |
| 403 | `FORBIDDEN`、`PROJECT_ACCESS_DENIED`、`PROJECT_DISABLED`、`EMAIL_VERIFICATION_REQUIRED` | 不只是权限，还有项目禁用与待验证邮箱 |
| 404 | `RESOURCE_NOT_FOUND`、`CONTACT_/TEMPLATE_/CAMPAIGN_/WORKFLOW_NOT_FOUND` | 资源不存在或不属于本项目 |
| 409 | `CONFLICT`、`IDEMPOTENCY_KEY_REUSED` | 资源冲突 / 幂等键重放被拒 |
| 422 | `VALIDATION_ERROR`、`INVALID_EMAIL`、`MISSING_REQUIRED_FIELD` | schema 校验失败，读 `error.errors[]` |
| 429 | `RATE_LIMIT_EXCEEDED` | 指数退避 |
| 500 | `INTERNAL_SERVER_ERROR`、`DATABASE_ERROR`、`EXTERNAL_SERVICE_ERROR` | 记 requestId 后重试 |

**步骤**

1. **定 key**：目标端点是 `POST /v1/track` 且必须跑在浏览器/移动端 → 用 `pk_`；其余一律服务端用 `sk_`。完成标准：代码里不存在"前端引用 `sk_`"的路径，`sk_` 只从环境变量读取。
2. **装认证头**：`Authorization: Bearer <key>`，每次请求都带。完成标准：一次最小调用返回 2xx；若 401 转第 8 步。
3. **判断要不要幂等键**：重复执行会有真实副作用 → 生成一个 1–255 可打印 ASCII 的键，按"逻辑操作"唯一（UUID 或 `receipt-order-1234` 这类业务派生键）。完成标准：键可从业务数据稳定复算，且不会跨项目碰撞（键作用域是项目级，不是端点级）。
4. **处理多收件人粒度**：`/v1/send` 传数组时，键覆盖整次请求而非单个收件人。要按收件人控制 → 改成"一人一请求一键"。完成标准：不存在"一个键对应一批收件人且需要部分重试"的场景。
5. **读响应**：public API 先判 `success` 再取 `data`；dashboard 端点直接是资源对象（列表端点除外，列表是 `{data, cursor, hasMore, total}`）。完成标准：客户端解析按端点族分两套，且对 400 做过容错（见第 9 步的文档口径）。
6. **出错时按 `error.code` 分支**：`VALIDATION_ERROR` 读 `errors[]`（每项含 `field` 点路径如 `attachments.0.filename`、`message`、`code`、`received`）；`INVALID_API_KEY` 提示换 key；`RATE_LIMIT_EXCEEDED` 走指数退避 1s/2s/4s/8s；`BILLING_LIMIT_EXCEEDED` 排队或报错；`INTERNAL_SERVER_ERROR` 记 requestId 后稍等重试。完成标准：代码里没有任何分支是"只看 HTTP 状态码"的。
7. **失败后决定重试策略**：4xx → 修好请求后用**同一个**键重试；2xx/5xx → 键已占用，若确认 5xx 请求完全没生效则**换新键**重试。收到 409 → 读 `details.originalStatusCode`：非 null 表示首次已完成，null 表示并发竞争失败（稍后重试或视为已被另一个并发请求覆盖）。完成标准：重试路径有明确终止条件，不会无限循环。
8. **401 排查**：依次核对 key 前缀、复制是否带空格、是否用了 Bearer 格式、是否已被轮换。完成标准：定位到四项之一并修正。
9. **⚠️ 文档口径不一致，两者并列，不要替文档做取舍**：限流有两套口径——托管版 overview 说 "API requests — 1000 requests/minute per project"；自托管文档说限流默认关闭（`RATE_LIMIT_ENABLED=false`），开启后是 Redis token bucket（track 100/s burst 200、send 20/s burst 50、contacts 50/s burst 100），按项目而非 IP 计费，响应带 `RateLimit-Limit/Remaining/Reset`，429 带 `Retry-After`，Redis 不可达时放行而非拒绝。文档未说明两套实现的关系，自托管时先确认 `RATE_LIMIT_ENABLED` 的实际值再设计退避。
10. **⚠️ 文档口径不一致，两者并列，不要替文档做取舍**：`contacts/createContact`、`segments/createSegment`、`templates/createTemplate` 的 **400** 只返回 `{ error: "<string>" }`，没有 `success` / `error.code` / `requestId` / `timestamp`，与 errors 页"所有错误都遵循统一格式"矛盾（同端点的 401 又是标准信封）。客户端解析 400 时必须同时兼容两种形状，不能假定所有错误都带 `error.code`。
11. **取 requestId 并落日志**：从 `error.requestId` 或响应头 `X-Request-ID` 取值（成功响应也有该头），写进自己的应用日志。完成标准：任何一次异常调用都能从日志里反查到 requestId。
12. **分页遍历**：列表端点优先 `limit`（默认 20、上限 100，超 100 会被 clamp）+ `cursor`，翻到 `hasMore=false` 停止，`total` 只在首页读；`GET /campaigns`、`GET /templates`、`GET /segments/:id/contacts` 走 `page`/`pageSize` 页码分页（响应带 `total`/`page`/`pageSize`，`pageSize` 上限 100）；`GET /segments` 不分页直接返回裸数组。完成标准：遍历终止条件写成 `hasMore === false`（或页码版的 `page >= totalPages`），而不是"返回条数 < limit"。
13. **批量走队列**：`POST /contacts/lookup` 一次 ≤500 个邮箱；`POST /contacts/import` 文件 ≤5MB（multipart）；`POST /contacts/bulk-subscribe|bulk-unsubscribe|bulk-delete` 各 ≤1000 个联系人 ID。三者均异步并返回 `jobId`，用 `GET /contacts/bulk/:jobId` 或 `GET /contacts/import/:jobId` 轮询。完成标准：脚本轮询到 job 终态才结束，而不是发完请求就认为成功。
14. **密钥事件处置**：轮换 → 审计 Activity 窗口内的异常发送/联系人改动/campaign 变更 → 查 Billing → Consumption 用量尖峰 → 发现未授权活动时带 requestId 联系支持。轮换前先列全消费方清单（后端 env、前端 bundle、第三方集成）并安排发布窗口。完成标准：轮换后所有消费方在同一窗口内更新完毕，零残留 401。

15. **（可选）用 `/v1/verify` 做准入校验**：`POST /v1/verify`（需 `sk_`）检查邮箱格式、MX 记录、一次性域名、别名/转发、plus addressing、个人邮箱服务商、域名是否存在（NS）、是否 typo 并给出建议。响应十个维度：`valid`、`isDisposable`、`isAlias`、`isTypo`、`isPlusAddressed`、`isPersonalEmail`、`domainExists`、`hasWebsite`、`hasMxRecords`、`suggestedEmail`，外加人类可读的 `reasons[]`。完成标准：注册/导入前用它拦掉坏地址以降低退信；`hasWebsite` 文档明确标注"informational only"，不要拿它当准入判据。自托管可设 `VERIFY_EMAIL_ON_SIGNUP=true` 在注册时做同样的检查。

**输出契约**

产出：
1. 一份可运行的调用封装（按端点族分两套解析、按 `error.code` 分支、统一记录 `X-Request-ID`）。
2. 幂等键生成与重试策略说明（含 4xx/5xx/409 三条分支）。
3. 若涉及密钥事件：新的 key 对、影响面结论、可提交支持的 requestId 列表。

验收：
- 用 `pk_` 调 `/v1/send` 能稳定复现 401 且代码不会再犯；
- 同一幂等键二次提交返回 409 而非重复发送；
- 列表遍历能在 `hasMore=false` 处终止且不丢不重；
- 任取一次历史调用，能从日志里拿到 requestId。

## B 边界与失效

1. **不要用本卡去解决端点自身的业务语义**：本卡只讲调用契约。`/v1/send` 的必填组合、模板覆盖规则见 `plunk-transactional-send`；`data.*` 与保留键见 `plunk-contacts-and-fields`。
2. **幂等键不是长周期去重**：键 24 小时后过期并可被再次使用，跨天补偿任务会真正再发一次。文档未给出长周期去重方案（未说明如何做），业务侧应自行持久化"已发送"状态；自托管可调 `IDEMPOTENCY_KEY_TTL_HOURS`，但文档提示会让 `idempotency_keys` 表等比增长。
3. **409 不等于"拿到上次结果"**：它是"拒绝"而非"重放"。需要 `emailId` 做 webhook 关联的系统必须在首次成功响应里落库，别指望 409 补回来。
4. **密钥轮换无宽限期**：把轮换当成"新旧并存平滑过渡"会导致所有未更新的消费者同时 401。多个消费方时必须有发布窗口。
5. **`pk_` 只能调 `/v1/track`**：任何"为了省事把 `sk_` 下发到前端"的做法都会把整个 API 面暴露出去；反过来 `pk_` 也不应被当作完全无害——文档要求视其为半敏感，被滥用也要轮换。
6. **⚠️ 文档口径不一致，两者并列，不要替文档做取舍**：账户级与部分资源级操作没有 API。域名增删、计费、项目删除、密钥轮换等都需要 dashboard 会话（MCP 也刻意不暴露），不要去猜不存在的端点。
7. **⚠️ 高危不可撤销：`POST /users/@me/projects/:id/reset`**（"Wipe project data (irreversible)"）会清空项目全部联系人、模板、活动、workflow 且不可恢复；`DELETE /users/@me/projects/:id` 直接删项目。不要把 reset 当作"清空测试数据重新来过"的常规操作；需要隔离时用独立的 staging 项目（文档明确建议 staging 与 production 分项目，使泄露的 staging key 无法触及生产数据）。
8. **`total: 0` 不是总数为零**：游标分页第 2 页起的 `total` 恒为 0，是"未计算"。把后续页的 `total` 当总数会得出错误结论。
9. **403 有多重语义**：除了权限不足，还包含 `PROJECT_DISABLED`（项目被禁用）与 `EMAIL_VERIFICATION_REQUIRED`（需要验证邮箱）。只看状态码会误导排障方向——发信域名未验证导致的 403 应转 `plunk-domain-deliverability`。
10. **限流口径不可混用**：托管与自托管是两套描述，自托管在 `RATE_LIMIT_ENABLED=false` 时不会拒绝任何流量（升级也不会突然开始拒绝）。设计退避前先确认实际部署形态。
