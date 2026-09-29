# 数据保留

## R 原文依据

- [/guides/data-retention](https://docs.useplunk.com/guides/data-retention)："Email metadata and metrics — Indefinitely — Never removed. Rendered email HTML — 90 days — Body is cleared; the email row remains. API request logs — 30 days — Log entry is deleted. Idempotency keys — 24 hours — Key is deleted and becomes reusable."
- [/guides/data-retention](https://docs.useplunk.com/guides/data-retention)："Clearing is irreversible. If you need a permanent archive of exact sent content, capture it on your side at send time, or reconstruct it from the template and the contact's data."
- [/guides/data-retention](https://docs.useplunk.com/guides/data-retention)："The only visible effect is that emails older than 90 days no longer offer a preview of their content in the dashboard. Segment filters built on email activity are unaffected, since they read the event timestamps rather than the body."
- [/guides/data-retention](https://docs.useplunk.com/guides/data-retention)："Plunk logs most API requests — public API calls, resource changes, authentication attempts, and any failed request... These log entries are deleted after 30 days. This affects request-level troubleshooting only; the emails and events those requests created are unaffected."
- [/guides/data-retention](https://docs.useplunk.com/guides/data-retention)："Retention runs as scheduled jobs on the worker process, so a deployment that never runs the worker will grow without bound."
- [/guides/data-retention](https://docs.useplunk.com/guides/data-retention)："Email body cleanup: Daily, 04:00. API request log cleanup: Daily, 03:00. Idempotency key cleanup: Hourly, on the hour. Daily jobs run in the worker container's local timezone. Set `TZ` on that container if you need them to land at a specific hour."
- [/guides/data-retention](https://docs.useplunk.com/guides/data-retention)："The 90-day and 30-day windows are compiled in rather than read from the environment. The idempotency key TTL is the exception and can be tuned with `IDEMPOTENCY_KEY_TTL_HOURS`."
- [/self-hosting/environment-variables](https://docs.useplunk.com/self-hosting/environment-variables)："`IDEMPOTENCY_KEY_TTL_HOURS` — How long a used key stays claimed before it can be reused. Expired keys are swept hourly, so raising this grows the `idempotency_keys` table proportionally. Default `24`."

## I 解释

Plunk 把"记录"和"记录上挂着的大块数据"分开对待：前者是分析、分群、计费的对象，永久保存；后者是成本和存储压力的主要来源，按时间表老化。这条边界是本卡的判断核心——**凡是要拿来分析的东西都不会被清掉，凡是只能用来"回看"的东西都会过期**。

第一层是保留窗口表。邮件元数据与指标（收件人、主题、发件人、发送时间、投递状态、投递/打开/点击/退信/投诉的时间戳、打开与点击计数、来源 campaign/workflow/template）**永久**保留；渲染后的 HTML 正文 **90 天**后被清空——注意是清空 body 而不是删行，email 记录本身还在；API 请求日志 **30 天**后删除；幂等键 **24 小时**后删除并可复用。

| 数据 | 保留时长 | 到期后发生什么 | 影响面 |
| --- | --- | --- | --- |
| 邮件元数据与指标 | 永久 | 永不移除 | 统计、分群、计费全部照常 |
| 渲染后的 HTML 正文 | 90 天 | body 被清空，email 行保留 | 只影响 dashboard 的 Preview；基于 email activity 的 segment 不受影响 |
| API 请求日志 | 30 天 | 日志条目被删除 | 只影响请求级排障；请求创建的邮件与事件不受影响 |
| 幂等键 | 24 小时 | 键被删除并变为可复用 | 过期是幂等机制的一部分，不是存储清理 |

自托管侧的作业调度（均跑在 worker 进程上）：

| 作业 | 调度 |
| --- | --- |
| 邮件正文清理 | 每天 04:00 |
| API 请求日志清理 | 每天 03:00 |
| 幂等键清理 | 每小时整点 |

第二层是为什么要这样设计。正文是单封邮件里体积最大的部分（联系人动态流里 Preview 按钮显示的就是它），所以它是唯一被主动清掉的东西；而清空只影响"能不能预览"，不影响任何分析维度——基于 email activity 的 segment 过滤读的是事件时间戳而非 body，所以完全不受影响。幂等键那一行容易被误读成存储清理，其实相反：**过期正是键变得可复用的原因，清理是幂等机制的一部分**。

第三层是不可逆性带来的义务。90 天后正文永久不可恢复，所以有合规、审计、客诉取证需求时，必须在**发送侧自己落库**（或事后用模板 + 当时的联系人 data 重建，但重建结果可能与原文不一致，因为模板和 data 都可能已变）。把 Plunk 当作发送内容的唯一存档是一个会真实踩到的坑。

第四层是自托管的执行前提。保留是跑在 **worker 进程**上的定时任务，所以只部署 `SERVICE=api`、或容器编排里漏了 worker 的部署，清理任务永不执行，数据无界增长。每日任务用 worker 容器的**本地时区**，想固定执行时刻就必须在 worker 容器上设 `TZ`。

第五层是可配置性的边界：90 天与 30 天是编译进代码的常量，自托管也改不了；只有幂等键 TTL 是例外，可用 `IDEMPOTENCY_KEY_TTL_HOURS` 调（默认 24），调高会让 `idempotency_keys` 表等比增长。

第六层是升级的代价。升级后的首次运行会**一次性**清掉所有超过 90 天的正文积压，大库耗时长；而且 Postgres 会把释放的空间留给新行，**磁盘不会自动收缩**，真要回收磁盘必须显式 `VACUUM FULL`。

## A1 示例

原书未提供本主题的 worked example，以下为按 `/guides/data-retention` 规则可复算的合成演练。

**示例 1 · 90 天后要做客诉取证**（合成演练）

- 输入/问题：客服要调出 6 个月前发给某客户的一封营销邮件的实际渲染内容。
- 怎么执行：先确认该邮件发送时间距今已超 90 天 → 正文已被清空且不可逆。只能二选一：① 当时在发送侧留存了内容则查自家归档；② 用当时的模板 + 该联系人当前的 data 重建。
- 产出/预期输出：email 行、收件人、主题、发送时间、投递与打开点击时间戳**都还在**；正文**永久不可恢复**。重建结果可能与原文不一致（模板与 data 都可能已变更），文档未说明重建的保真度。

**示例 2 · 一个月后回头排查一次失败请求**（合成演练）

- 输入/问题：35 天前某次 `/v1/send` 返回异常，现在想查当时的请求详情。
- 怎么执行：Plunk 侧请求日志已按 30 天窗口删除，只能查自己调用侧留的记录——响应头 `X-Request-ID` + 请求/响应快照。
- 产出/预期输出：Plunk 日志侧无线索；邮件与事件本身**不受影响**（仍在）。结论是调用侧必须从第一天起自行记录 requestId 与快照。

**示例 3 · 只跑 api 服务的自托管部署**（合成演练）

- 输入/问题：为省资源把容器拆成 `SERVICE=api` + `SERVICE=web`，没有 worker。
- 怎么执行：三个清理作业（正文 04:00、请求日志 03:00、幂等键整点）都跑在 worker 上 → 一次也没执行过。
- 产出/预期输出：正文、日志、幂等键无限堆积，存储成本失控。修法是让 worker 常驻（或 `SERVICE=all`），并在 worker 容器上设 `TZ` 固定执行时刻。

**示例 4 · 大库升级后的首次清理**（合成演练）

- 输入/问题：一个跑了两年、体量很大的自托管实例准备升级。
- 怎么执行：预期首次运行会把**所有**超过 90 天的正文一次性清掉，耗时很长；安排在业务低峰。清理后看磁盘占用。
- 产出/预期输出：数据被清掉，但磁盘占用**不下降**——Postgres 会把空间留给新行。要真正收缩磁盘必须 `VACUUM FULL`。⚠️ 文档未给出 `VACUUM FULL` 的副作用与耗时说明，需自行验证后再执行。

## A2 何时会用

**用户场景**

1. 要做合规审计或客诉取证，想回看某封历史邮件的实际发送内容。
2. 排查一个"很久以前"的 API 请求，发现 Plunk 侧日志已经没了。
3. 自托管部署（尤其拆分 `SERVICE` 或自定义编排）时要确认清理作业会不会真的执行。
4. 大库自托管实例准备升级，要评估首次清理的耗时与磁盘表现。
5. 想调整某个保留窗口（例如把正文保留得更久、或把幂等键 TTL 调长）。

**语言信号（中英双写）**

`数据保留` / `data retention` / `90 天` / `90 days` / `30 天` / `API request logs` / `幂等键` / `Idempotency-Key` / `24 hours` / `正文被清空` / `body is cleared` / `不可逆` / `irreversible` / `归档` / `archive` / `worker` / `TZ` / `IDEMPOTENCY_KEY_TTL_HOURS` / `VACUUM FULL` / `preview`

**一句话判据**：需要"回看内容"的诉求（正文预览、请求日志）都会过期；需要"拿来算"的诉求（统计、分群、计费）都不过期。

**与相邻能力的区分**

- 本卡负责：保留窗口、清理作业的调度与前提、不可逆后果与自建归档义务。
- `plunk-api-essentials` 负责幂等键的**语义**（2xx/5xx 保留、4xx 释放、409 冲突）；本卡只负责它的生命周期与 TTL 可调这一事实。
- `plunk-self-hosting` 负责容器编排、`SERVICE` 拆分与环境变量总表；本卡只取"清理跑在 worker 上、时区用 `TZ`"这一条。
- `plunk-campaigns-analytics` 与 `plunk-segments-filters` 负责统计与分群口径；本卡只确认它们读的数据（事件时间戳）不在被清理之列。
- `plunk-billing-limits` 负责用量与上限；保留与计费互不影响（正文被清掉的邮件仍计入历史用量）。

**按诉求快速分流**

- "三个月前的邮件内容还能不能看" → 本卡。
- "这个幂等键还能不能复用、要不要重试" → `plunk-api-essentials`。
- "worker/容器/环境变量怎么配" → `plunk-self-hosting`。
- "这个月的用量与上限" → `plunk-billing-limits`。

## E 执行步骤

**输入契约**

必需信息：
1. 诉求类型：回看内容、排查请求、自托管运维、还是调整保留参数。
2. 部署形态：托管版（useplunk.com）还是自托管；自托管还需 worker 是否常驻、容器 `TZ`。
3. 涉及数据的时间点（距今天数），用来对照 90 / 30 / 24h 三档。
4. 是否有合规/取证要求（决定要不要自建归档）。

缺失时先问：是不是自托管、worker 有没有在跑、要看的那封邮件距今多久。

**输出契约**

产出：保留影响面判定（哪些数据还在、哪些已不可逆丢失）、自建归档方案（发送侧落库或模板重建）、自托管作业检查清单（worker + `TZ` + 升级窗口）、以及一份磁盘回收预期说明。验收：① 每一类数据都能说出落在 90/30/24h/永久 哪一档；② 有合规需求时归档方案已落到发送侧；③ 自托管能证明 worker 常驻且清理作业有预期执行时刻。

1. **先判档位**：对照保留窗口表——元数据与指标永久；渲染 HTML 正文 90 天（清空 body，行保留）；API 请求日志 30 天；幂等键 24 小时。完成标准：手上每一类数据都有明确档位，不含糊说"应该还在"。
2. **判断影响面**：若诉求只是统计/分群/计费 → 不受影响（读的是事件时间戳而非 body），直接结束；若诉求是看内容 → 转第 3 步。完成标准：能明确说出"唯一可见影响是 90 天以上的邮件在 dashboard 上不再有 Preview"。
3. **需要永久存档就在发送侧留**：正文清空不可逆，合规/审计/取证需求必须在发送时自己落库；事后重建只能用模板 + 当时的联系人 data，且结果可能与原文不一致。完成标准：归档动作发生在**发送时**，而不是等要取证时才发现没留。
4. **请求级排障线索自己记**：调用侧记录响应头 `X-Request-ID` 与请求/响应快照，不要指望 30 天后再来查 Plunk 日志。完成标准：日志留存责任在自己侧，且邮件与事件本身不受日志删除影响这点已确认。
5. **自托管确认 worker 常驻**：保留作业跑在 worker 进程上，`SERVICE=api` 等不跑 worker 的部署会让数据无界增长。完成标准：编排里存在常驻 worker（或 `SERVICE=all`），且能用进程/日志证明它在跑。
6. **需要固定执行时刻就设 `TZ`**：正文清理每天 04:00、请求日志每天 03:00、幂等键每小时整点，均按 worker 容器本地时区。完成标准：worker 容器上显式设置了 `TZ`，且知道这三个时刻的含义。
7. **调参只调幂等 TTL**：90/30 天是编译期常量，自托管也无法改；只有 `IDEMPOTENCY_KEY_TTL_HOURS`（默认 24）可调。完成标准：接受 90/30 不可配这一事实，且知道调高 TTL 会让 `idempotency_keys` 表等比增长。
8. **升级前规划维护窗口**：首次运行会一次性清空所有超期正文积压，大库耗时长，安排在业务低峰。完成标准：升级计划里包含"首次清理窗口"这一项。
9. **磁盘预期要说清**：清理后 Postgres 会复用释放的空间给新行，表在磁盘上不会自动收缩；要回收磁盘必须 `VACUUM FULL`。完成标准：向干系人说明"数据少了但磁盘不降"是预期行为，不是清理失败。
    - 若诉求是"看内容且已超 90 天"：直接告知不可逆（第 3 步），不要给出任何恢复方案。
    - 若是自托管且没跑 worker：第 5 步是阻断项，先修编排再谈其他。
    - 若准备升级大库：先做第 8–9 步，再执行升级。
    - ⚠️ 文档口径不一致，两者并列，不要替文档做取舍：保留窗口的可配置性边界——`/guides/data-retention` 说 90/30 天窗口"编译进代码、不从环境读取"，只有幂等键 TTL 可用 `IDEMPOTENCY_KEY_TTL_HOURS` 调；而自托管环境变量页把 `IDEMPOTENCY_KEY_TTL_HOURS` 描述为"可调，默认 24"，却未说明托管版是否同样可配、也未说明 90/30 在托管版是否绝对不可配。按"自托管可配幂等 TTL、90/30 一律不可配"执行，其余部分文档未说明，需自行验证。

## B 边界与失效

1. **不要等过期了才想起归档**：90 天后正文永久不可恢复，这是本卡唯一的 0 recoverability 操作。有合规/取证需求必须从第一次发送就开始在自家侧留存。
2. **不要用"Preview 没了"推断"邮件没了"**：清空的是 body，email 行与全部元数据/指标都还在，统计与分群不受影响。
3. **不要指望 30 天后还能查请求日志**：请求级排障线索只保留 30 天，延迟排查必然落空；调用侧要自己记 `X-Request-ID`。
4. **不要把幂等键过期当成存储问题**：过期是键可复用的前提，是幂等机制的一部分；调高 TTL 不只是"多存一会儿"，会等比放大 `idempotency_keys` 表。
5. **不跑 worker 的自托管等于没有保留**：清理任务永不执行、数据无界增长。这是自托管拆分部署时最容易漏掉的一项。
6. **磁盘不会自动变小**：清理后 Postgres 复用空间给新行，表在磁盘上不收缩；`VACUUM FULL` 才是回收手段，但文档未给出其副作用与耗时，需自行验证后再执行（见 `plunk-self-hosting`）。
7. **90/30 天不可配**：不要花时间找配置项或试图用环境变量改；只有幂等 TTL 可配。托管版是否同样适用，文档未说明。
8. **本卡不负责幂等语义**：2xx/5xx 保留键、4xx 释放键、409 冲突怎么处理属于 `plunk-api-essentials`；本卡只说键多久后被删。
9. **高危/不可逆操作**：邮件正文 90 天清空（不可逆，无恢复路径，只能靠发送侧自建归档兜底）；`VACUUM FULL`（文档未说明副作用，执行前需自行验证并备份）。
