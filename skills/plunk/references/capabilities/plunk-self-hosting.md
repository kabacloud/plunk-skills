# 自托管部署与运维

## R 原文依据

- [/self-hosting/introduction](https://docs.useplunk.com/self-hosting/introduction)："Requirements: Docker and Docker Compose; AWS SES account (for sending emails); Domain name with subdomains for each service. app → Dashboard, api → API, docs → Documentation, www → Landing page, smtp → SMTP relay (optional)."
- [/self-hosting/docker](https://docs.useplunk.com/self-hosting/docker)："80 Nginx (HTTP) — fronts API, dashboard, landing, wiki. 465 SMTP (implicit TLS). 587 SMTP (STARTTLS). 8080 API server (proxied through nginx). 9000 Minio API. 9001 Minio web console. Postgres, Redis, and Minio are kept internal to the Docker network by default."
- [/guides/data-retention](https://docs.useplunk.com/guides/data-retention)："Retention runs as scheduled jobs on the worker process, so a deployment that never runs the worker will grow without bound."
- [/self-hosting/email-setup](https://docs.useplunk.com/self-hosting/email-setup)："By default, new AWS SES accounts are in sandbox mode: Send only to verified addresses, 200 messages per 24 hours, 1 message per second... AWS typically approves within 24 hours."
- [/self-hosting/email-setup](https://docs.useplunk.com/self-hosting/email-setup)："Plunk does not auto-provision receipt rules, so this is a manual one-time setup in your AWS account. Not all AWS regions support SES inbound — confirm inbound-smtp.<your-region>.amazonaws.com exists for your AWS_SES_REGION before continuing."
- [/self-hosting/environment-variables](https://docs.useplunk.com/self-hosting/environment-variables)："Rate limiting is off by default — upgrading Plunk will never start refusing traffic your callers were already sending. Set RATE_LIMIT_ENABLED=true to switch it on; none of the other variables have any effect until you do. If Redis is unreachable, requests are allowed through rather than rejected."
- [/self-hosting/environment-variables](https://docs.useplunk.com/self-hosting/environment-variables)："Projects are automatically disabled if either: 1. A single email is detected with confidence ≥ PHISHING_CONFIDENCE_THRESHOLD, or 2. PHISHING_CUMULATIVE_THRESHOLD or more emails are flagged within the PHISHING_CUMULATIVE_WINDOW_MS time window."
- [/self-hosting/environment-variables](https://docs.useplunk.com/self-hosting/environment-variables)："All Stripe variables must be set together for billing to activate... If you use a Stripe Radar rule to request 3D Secure, exclude off-session payments from it (and not :is_off_session:). The verification charge runs with no cardholder present."

## I 解释

自托管不是"把托管版搬到自己机器上"，而是**你要接管一条托管版由 Plunk 运维的链路**：SES 凭据与事件回传、后台作业进程、限流、清理任务、计费。这一卡的方法论是"先画清责任边界，再逐项配"，因为绝大多数自托管故障不是代码问题，而是某一环没人负责。

第一层是**形态与端口**。compose 里是五个服务：`plunk`（API + dashboard + landing + wiki + SMTP，前面挂 nginx）、`postgres`（PG16）、`redis`（Redis 7）、`minio`（S3 兼容存储）、`ntfy`（系统通知）。对外只需要暴露 80（用 SMTP 再加 465/587），8080 是 API 自身的端口、被 nginx 反向代理，9000/9001 是 Minio——后三者默认只留在 Docker 网络内。要拆进程时用 `SERVICE=api | worker | web | all`，探活用 `GET /health`，返回 `{status, time, uptime}`，API 进程能服务时给 200。

第二层是**worker 不能省**。保留清理（邮件正文 90 天、请求日志 30 天、幂等键每整点）是 worker 上的定时任务，不跑 worker 的部署会无界增长；同时还会失去 segment 重算、snooze 回归等后台作业。所以"只跑 api 省资源"是一个会累积成磁盘事故的取舍，而不是优化。

第三层是**SES 侧的八个动作**。IAM 只需七个 action（`ses:SetIdentityMailFromDomain`、`GetIdentityDkimAttributes`、`SendRawEmail`、`GetIdentityVerificationAttributes`、`VerifyDomainDkim`、`ListIdentities`、`SetIdentityFeedbackForwardingEnabled`，Resource `*`），因为 Plunk 运行时不调用 SES 收信 API——收信的 receipt rule 是 AWS 管理员的一次性手工活，包括建 rule set、填 recipient condition、动作发布到 SNS、设为 active，再给每个要收信的已验证域加区域化 MX。事件回传走一个 Standard SNS 主题订阅 `https://api.<你的域>/webhooks/sns`，Plunk 自动确认订阅（失败时去日志里找确认 URL）。两个 configuration set 的差别只在 Opens/Clicks：`plunk-tracking` 含 Send/Delivery/Open/Click/Bounce/Complaint，`plunk-no-tracking` 只有前四个加后两个；**不配第二个集合，项目里的追踪开关会被隐藏**。

第四层是**URL 不要手填**。应用从 `*_DOMAIN` 环境变量在容器启动时自动推导所有内部与客户端 URL，所以不需要（也不应该）手工设 `*_URI` 或 `NEXT_PUBLIC_*`。同理，存储与通知也是开箱即用的：Minio 自带 S3 兼容桶（`S3_BUCKET` 默认 `uploads`、`S3_FORCE_PATH_STYLE=true`），ntfy 自带一个自托管实例（`NTFY_URL` 默认 `http://ntfy/plunk-notifications`）；只有 `PLUNK_API_KEY` + `PLUNK_FROM_ADDRESS` 配了，平台通知才会以邮件形式发出，否则只有 ntfy 通知。容器时区由 `TZ` 决定，它直接决定每日清理作业落在哪个小时。

第五层是**几处"默认值会咬人"的配置**：① 新 SES 账号在沙箱里（只能发已验证地址、200 封/24h、1 封/秒），生产必须出沙；② MAIL FROM 子域默认前缀 `plunk`，若该子域已被 R2/CDN 的 CNAME 占用，MX+TXT 与 CNAME 无法共存，只能换前缀；③ SMTP 不挂证书就以无 TLS 运行（不是起不来），acme.json 方式必须配 `SMTP_DOMAIN` 才能在多证书里挑对那张；④ 限流默认关闭，只在暴露给不可信客户端时才开；⑤ 反钓鱼检测与 `AUTO_PROJECT_DISABLE` 都会自动停用项目，停用时发送全部 403、入站也不再处理。

## A1 示例

**示例 1 · 最小 .env 落到可服务的实例**（来自 `/self-hosting/docker` 的 Minimal .env 与 Quick Start）

- 输入/问题：一台装好 Docker Compose 的服务器 + 一个域名 + AWS SES 凭据，想最快跑起来。
- 怎么执行：写最小 .env（`JWT_SECRET="$(openssl rand -base64 32)"`、`DB_PASSWORD`、四个 `*_DOMAIN`、`USE_HTTPS="true"`、`AWS_SES_REGION/ACCESS_KEY_ID/SECRET_ACCESS_KEY`、`SES_CONFIGURATION_SET="plunk-tracking"`）→ `docker compose up -d` → `curl https://api.yourdomain.com/health`。
- 产出/预期输出：返回 `{"status":"ok","time":...,"uptime":...}` 与 200；文档未说明其余字段的确切类型。反向代理后只需暴露 80（用 SMTP 再加 465/587）。

**示例 2 · 只跑 api 不跑 worker 的后果**（来自 `/guides/data-retention` 的 worked example 与 `/self-hosting/docker` 的 SERVICE 表）

- 输入/问题：为省资源给容器设了 `SERVICE=api`，几周后磁盘持续增长。
- 怎么执行：确认 compose 里是否存在 `SERVICE=worker`（或 `SERVICE=all`）的进程；保留作业（正文每天 04:00、请求日志每天 03:00、幂等键每小时整点，时区由容器 `TZ` 决定）只在 worker 上跑。
- 产出/预期输出：没有 worker 时清理任务永不执行，数据无界增长，同时失去 segment 重算等后台作业；补上 worker 后首次运行会一次性清掉全部超期积压（大库会很慢），且 Postgres 不会自动收缩表（需 `VACUUM FULL`）。

**示例 3 · MAIL FROM 默认子域被 CDN 占用**（来自 `/self-hosting/environment-variables` 的 `MAIL_FROM_SUBDOMAIN` 说明）

- 输入/问题：想配自定义 MAIL FROM 改善 DMARC 对齐，但 `plunk.yourdomain.com` 已经是 R2/CDN 的自定义域名。
- 怎么执行：不要试图加记录——MAIL FROM 主机名需要 MX + TXT，与已存在的 CNAME 不能共存；改用 `MAIL_FROM_SUBDOMAIN` 换一个未被占用的前缀。
- 产出/预期输出：换前缀后可按 SES 显示的 MX + TXT 完成配置；沿用默认前缀则 DMARC 对齐做不了。IAM 策略里已包含 `ses:SetIdentityMailFromDomain`，无需额外授权。

**示例 4 · 入站收信在不支持 inbound 的 region 上配不出来**（来自 `/self-hosting/email-setup` 的 inbound 小节）

- 输入/问题：按文档建好身份并加了 MX，入站邮件始终收不到。
- 怎么执行：先确认 `inbound-smtp.<your-region>.amazonaws.com` 对你的 `AWS_SES_REGION` 存在；存在则在 SES 控制台建/选 rule set，填 recipient condition `*@yourdomain.com`，动作设为 Publish to SNS，并把 rule set 设为 active；不存在则出站留在当前 region，另在支持 inbound 的 region 建独立身份。
- 产出/预期输出：Plunk 不会自动建 receipt rule，这一步只能人工做；区域不支持时 MX 指向会静默无效。文档未给出验证收信是否打通的具体端点。

## A2 何时会用

**用户场景**

1. 第一次把 Plunk 装到自己的服务器/VPS 上，需要一份能照抄的 .env、端口与子域规划。
2. 实例能起来但邮件发不出去，或发送量卡在每天 200 封——需要判断是不是 SES 沙箱、configuration set 或 SNS 订阅没配对。
3. 想启用入站收信，或已在 AWS 侧配了一堆但收不到信。
4. 磁盘/数据库持续增长、限流到底开没开、SMTP 端口是否加密这类运维排障。
5. 自托管上要开计费（Stripe），或开了 AI 反钓鱼后项目被莫名停用。

**语言信号**

`自托管` / `self-host` / `self-hosting` / `docker compose` / `docker-compose.yml` / `环境变量` / `environment variables` / `.env` / `SES` / `sandbox` / `出沙` / `production access` / `configuration set` / `SNS` / `receipt rule` / `inbound` / `MAIL FROM` / `SMTP` / `TLS 证书` / `/health` / `healthcheck` / `worker` / `SERVICE` / `rate limit` / `限流` / `phishing` / `钓鱼检测` / `AUTO_PROJECT_DISABLE` / `Stripe` / `Radar` / `3DS`

**与相邻能力的区分**

- 本卡负责：自托管实例的**部署形态、AWS 侧一次性配置、环境变量、进程拆分、探活与运维开关**。
- `plunk-domain-deliverability` 负责发信域的 DNS 记录本身（SPF/DKIM/DMARC/MAIL FROM 记录怎么填）。
- `plunk-data-retention` 负责保留窗口的口径（90/30/24h）与清理作业调度；本卡只负责"worker 必须跑"这一条运维前提。
- `plunk-inbound-email` 负责入站邮件在 Plunk 内的处理与 verdict；本卡只负责 AWS 侧 receipt rule 与 MX 这一段。
- `plunk-api-essentials` 负责 API 侧的错误码与限流响应头语义；本卡负责限流**开关与桶参数**的配置。
- `plunk-billing-limits` 负责托管版的计费口径与限额；本卡只负责自托管 Stripe 激活条件与 3DS 例外。

## E 执行步骤

**输入契约（必需信息）**

- 服务器是否已装 Docker 与 Docker Compose；域名及可用的 `app` / `api` / `www` / `docs` / `smtp` 子域。
- AWS 账号所在 region、SES 是否仍在沙箱、是否要启用入站收信。
- 是否要开计费（Stripe）、是否要用 AI 反钓鱼检测、是否对外暴露不可信 API 客户端。
- 缺失时先问：域名与四个子域、SES region、要不要收信、要不要 billing；不知道时给出默认值并标注需确认。

**输出契约（产出与验收）**

- 一份可直接落地的 `.env` 片段 + compose 服务/端口说明 + AWS 侧动作清单。
- 验收：`curl https://api.<域>/health` 返回 200 与 `{status, time, uptime}`；存在 `SERVICE=worker` 或 `all` 的进程；需要追踪时两个 configuration set 均已配置。

**步骤**

1. **确认前置条件**：Docker 与 Docker Compose 就绪、AWS SES 账号可用、域名可加四个子域。完成标准：四条子域都能解析到服务器，`USE_HTTPS="true"`（TLS 终结反代后面）已确定。
2. **规划子域并写最小 .env**：`API_DOMAIN` / `DASHBOARD_DOMAIN` / `LANDING_DOMAIN` / `WIKI_DOMAIN`，`JWT_SECRET` 用 `openssl rand -base64 32` 生成，`DB_PASSWORD` 自定。完成标准：应用能从 `*_DOMAIN` 自动推导所有内部与客户端 URL——**不要**手工设 `*_URI` 或 `NEXT_PUBLIC_*`。最小可用片段（来自 `/self-hosting/docker`）：

   ```bash
   # Required
   JWT_SECRET="$(openssl rand -base64 32)"
   DB_PASSWORD="your-secure-password"

   # Domains (point these to your server)
   API_DOMAIN="api.yourdomain.com"
   DASHBOARD_DOMAIN="app.yourdomain.com"
   LANDING_DOMAIN="www.yourdomain.com"
   WIKI_DOMAIN="docs.yourdomain.com"
   USE_HTTPS="true"

   # AWS SES
   AWS_SES_REGION="us-east-1"
   AWS_SES_ACCESS_KEY_ID="your-key"
   AWS_SES_SECRET_ACCESS_KEY="your-secret"
   SES_CONFIGURATION_SET="plunk-tracking"
   ```

3. **建 IAM 用户与最小策略**：用户 `plunk-ses`，Action 限定七个 ses action，Resource `*`。完成标准：策略只含这七个 action；收信的 receipt rule 不在此策略职责内（Plunk 运行时不调 SES 收信 API）。
4. **建 SNS 主题与订阅**：Standard 主题（如 `plunk-ses-events`），协议 HTTPS，Endpoint `https://api.<你的域>/webhooks/sns`。完成标准：Plunk 自动确认订阅；若失败，去日志里找确认 URL 手动确认。
5. **建两个 configuration set**：`plunk-tracking`（Sends/Deliveries/Opens/Clicks/Bounces/Complaints）与 `plunk-no-tracking`（Sends/Deliveries/Bounces/Complaints），事件目的地均指向 SNS 主题，并写入 `SES_CONFIGURATION_SET` / `SES_CONFIGURATION_SET_NO_TRACKING`。完成标准：两个集合都存在且 env 都填了；**只填第一个则项目里的追踪开关被隐藏**（若转第 4 步查 SNS 未确认也会表现为事件缺失）。
6. **在 dashboard 加域并补 DNS**：DKIM 由 Plunk 自动处理，按 dashboard 给出的值加记录。完成标准：域在 dashboard 显示为已验证（具体记录清单见 `plunk-domain-deliverability`）。
7. **判断沙箱状态**：若 SES 账号仍在沙箱（只能发已验证地址、200 封/24h、1 封/秒），生产必须走 Account dashboard → Request production access。完成标准：能用未验证地址正常发信；未出沙却上生产是明确的失败模式。
8. **（按需）配入站**：先确认 `inbound-smtp.<region>.amazonaws.com` 存在 → 建/选 rule set、recipient condition `*@yourdomain.com`、动作 Publish to SNS、设为 active → 为每个要收信的已验证域加 MX `10 inbound-smtp.<region>.amazonaws.com`。完成标准：rule set 为 active 且 MX 已生效；region 不支持时改为"出站留当前 region + 收信身份建在支持的 region"。
9. **（按需）配 MAIL FROM**：默认前缀 `plunk`；若该子域已被 R2/CDN 的 CNAME 占用，用 `MAIL_FROM_SUBDOMAIN` 换前缀。完成标准：子域上 MX + TXT 均已添加且不与既有 CNAME 冲突。
10. **（按需）配 SMTP 证书**：Traefik 场景挂 `acme.json` 并设 `SMTP_DOMAIN`；否则挂 `/certs/privkey.pem` 与 `/certs/fullchain.pem`。完成标准：465/587 上 TLS 可用；**不挂任何证书时 SMTP 以无 TLS 方式运行**，不会报错。
11. **配探活与进程拆分**：`GET /health` 接进 liveness/readiness 探针；需要拆进程时用 `SERVICE=api|worker|web|all`。完成标准：编排器能通过 `/health` 判定存活，且**至少有一个 worker 进程**——否则保留作业与后台作业永不执行（转第 12 步）。
12. **决定限流姿态**：⚠️ 文档口径不一致，两者并列，不要替文档做取舍：API overview 说"API requests 1000 requests/minute per project"；自托管文档说限流**默认关闭**（`RATE_LIMIT_ENABLED=false`），开启后是 Redis 里按项目（非 IP）的 token bucket——`/v1/track` 100/s + burst 200、`/v1/send` 20/s + burst 50、contacts 写操作 50/s + burst 100（POST/PATCH/DELETE 共享一个预算），单桶设 0 可单独关闭。完成标准：明确写下所处环境与所选口径；响应带 `RateLimit-Limit/Remaining/Reset`，超限返回 429 + `Retry-After`。另注意 **Redis 不可达时是放行而非拒绝**，所以网关层要另设兜底。
13. **决定安全类开关**：`AUTO_PROJECT_DISABLE` 默认 `true`（退信/投诉率超阈值自动暂停项目，停用时 403 且不再处理入站）；AI 反钓鱼需 `OPENROUTER_API_KEY` 才启用，命中"单封置信度 ≥ 95"或"1 小时内累计 ≥ 3 封"任一条件即自动停用项目。完成标准：要么接受自动停用并调参（采样率 0.1 / 阈值 95 / 累计 3 / 窗口 3600000ms），要么设 `AUTO_PROJECT_DISABLE=false` 改人工管理；文档未说明被自动停用后的恢复流程与通知方式。
14. **（按需）配 Stripe 计费**：`STRIPE_SK`、`STRIPE_WEBHOOK_SECRET`、`STRIPE_PRICE_ONBOARDING`、`STRIPE_PRICE_EMAIL_USAGE`、`STRIPE_METER_EVENT_NAME`（默认 `emails`）**必须齐全才激活**；若 Radar 有 3DS 规则，必须排除离场支付（`and not :is_off_session:`）。完成标准：订阅能完成激活；否则 1.00 的卡验证永远挂起、订阅被取消、入驻费全额退款。
15. **（可选）私有实例收口**：`DISABLE_SIGNUPS=true` 关闭注册；`VERIFY_EMAIL_ON_SIGNUP=true` 在注册时校验一次性域名、plus 寻址、域名存在性与 MX。完成标准：注册端点按预期拒绝/校验。

**环境变量默认值速查**（`/self-hosting/environment-variables`）

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `RATE_LIMIT_ENABLED` | `false` | 总开关，不开则其余限流变量全部无效 |
| `RATE_LIMIT_TRACK_PER_SECOND` / `_BURST` | `100` / `200` | `/v1/track` 桶 |
| `RATE_LIMIT_SEND_PER_SECOND` / `_BURST` | `20` / `50` | `/v1/send` 桶，另受 billing limit 约束 |
| `RATE_LIMIT_CONTACTS_PER_SECOND` / `_BURST` | `50` / `100` | contacts 写操作共享一个预算 |
| `MAX_ATTACHMENT_SIZE_MB` / `MAX_ATTACHMENTS_COUNT` | `10` / `10` | 40MB 是 SES 硬上限，不可配 |
| `IDEMPOTENCY_KEY_TTL_HOURS` | `24` | 调大会让 `idempotency_keys` 表按比例增长；过期键每小时清扫一次 |
| `PORT_SECURE` / `PORT_SUBMISSION` / `MAX_RECIPIENTS` | `465` / `587` / `5` | SMTP 端口与单封最多收件人数 |
| `MAIL_FROM_SUBDOMAIN` | `plunk` | 与 CNAME 冲突时必须换前缀 |
| `EMAIL_WORKER_MAX_CONCURRENCY` | `50` | 调高前要先放大 Prisma 连接池 |
| `NGINX_PORT` | `80` | 宿主机 80/443 被占用时改 |

## B 边界与失效

1. **不要用本卡替代域名 DNS 排障**。记录该加哪些、为什么只能有一条 SPF，属于 `plunk-domain-deliverability`；本卡只处理"谁负责加、加完在 Plunk 侧怎么验证"。
2. **不要假设限流已经在保护你**。自托管默认关闭，且 Redis 不可达时是 fail-open（放行而非拒绝）。把限流当硬性保护会在 Redis 故障时失去全部保护。
3. **不要只跑 `SERVICE=api`**。保留作业与后台作业都在 worker 上，长期不跑 worker 会无界增长；补上时首次清理是一次性大批量操作，大库会很慢且 Postgres 不自动收缩表。
4. **不要再想"让 Plunk 自动建 receipt rule"**。入站是 AWS 侧一次性手工动作，且并非所有 region 支持 `inbound-smtp.<region>.amazonaws.com`；区域不支持时只能拆成"出站一个 region + 收信身份另一个 region"。
5. **不要把沙箱当生产**。新 SES 账号只能发已验证地址、200 封/24h、1 封/秒；未出沙上生产表现为大面积发送失败且速率被拖垮。
6. **不要以为 SMTP 端口默认加密**。未挂载证书时 465/587 以无 TLS 运行且不报错；acme.json 方式漏配 `SMTP_DOMAIN` 会在多证书场景挑错证书。
7. **不要随手开 AI 反钓鱼而不看停用后果**。命中阈值即自动停用项目，发送全部 403 `PROJECT_DISABLED`，且入站也不再处理；文档未说明恢复流程与通知方式。
8. **不要在开了 Radar 3DS 的情况下接计费**。离场支付无法响应 3DS，验证永远挂起 → 订阅取消、入驻费全额退款。
9. **文档缺口（能力边界）**：备份 / 升级 / 回滚流程文档未给出；保留窗口的 90/30 天是硬编码、只有幂等 TTL 可调（见 `plunk-data-retention`）；`AUTO_PROJECT_DISABLE` 与反钓鱼的阈值以外行为（计数如何重置、抽样是否确定性）文档未说明。
10. **高危/难撤销操作**：SES 侧删除 configuration set 或 receipt rule set 会立即切断事件回传与收信；误把 `AUTO_PROJECT_DISABLE` 关掉会失去自动抑制保护；`VACUUM FULL` 与升级后首次批量清理都是长锁/不可逆的资源操作。项目级 reset 同样不可逆（见 `plunk-api-essentials`）。
11. **不要把平台通知邮件与发信混为一谈**。`PLUNK_API_KEY` + `PLUNK_FROM_ADDRESS` 只用于"项目被停用、限额触达"这类平台通知；它属于另一个项目，与业务发信的 SES 凭据不是一回事，也与 MCP 侧同名变量无关（见 `plunk-mcp-agents`）。
12. **不要在宿主机 80/443 已被占用时硬改 compose**。应改 `NGINX_PORT`（默认 `80`），让上层反代转发到别的端口；直接改容器内部映射会破坏 nginx 对 API / dashboard / landing / wiki 的统一前端。
