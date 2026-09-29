---
name: plunk-domain-deliverability
description: |
  当你要为新域名配置 Plunk 发信认证（DKIM 3×CNAME + SPF 1×TXT + bounce MX + DMARC，
  可选 MAIL FROM 子域），或域名已验证但邮件仍进垃圾箱、DNS 记录 24 小时仍未通过验证时使用。
  用户提供域名与当前 DNS 现状（是否已有 SPF、apex MX 指向谁），产出可直接照抄的记录清单、
  SPF 合并后的单条最终值、四步排障路径与症状→根因→动作速查表。
  不负责退信投诉的日常列表卫生（见 plunk-sending-health），也不负责收信后的处理（见 plunk-inbound-email）。
  triggers: 域名验证, DNS, SPF, DKIM, DMARC, MX, MAIL FROM, 垃圾箱, verify domain, deliverability, 403
metadata:
  cangjie.generated-by: cangjie-tools v2.5.0
  cangjie.capability-id: cap.plunk-docs.domain-and-dns
  cangjie.capability-revision: 1
  cangjie.bundle-id: bundle.plunk-docs
  cangjie.source-title: Plunk Developer Docs
  cangjie.tags: dns, deliverability, setup
---
# 发信域名与 DNS 认证

## R 原文依据

- [/guides/verifying-domains](https://docs.useplunk.com/guides/verifying-domains)："Domain verification requires adding several DNS records... **DKIM Records (3 CNAME records)**... **SPF Record (1 TXT record)**... **Bounce Handling (1 MX record)**... DMARC (optional, recommended)... MAIL FROM domain (optional)."
- [/guides/verifying-domains](https://docs.useplunk.com/guides/verifying-domains)："A domain can only have **one** SPF TXT record... you must **merge** Plunk's SPF mechanism into your existing record — don't add a second SPF record. Two separate SPF records will cause both to fail."
- [/guides/verifying-domains](https://docs.useplunk.com/guides/verifying-domains)："This MX record handles delivery feedback (bounces, complaints) for emails Plunk sends out — it's separate from the inbound MX record used to **receive** emails sent to your domain. They serve different purposes; you can have either one or both."
- [/guides/verifying-domains](https://docs.useplunk.com/guides/verifying-domains)："Start with `p=none` to monitor without affecting delivery. Once you've reviewed reports for a couple of weeks and confirmed all your legitimate senders are authenticated, tighten to `p=quarantine` and eventually `p=reject`."
- [/guides/verifying-domains](https://docs.useplunk.com/guides/verifying-domains)："If your DNS records aren't verifying after 24 hours: 1. Double-check the values... 2. Check DNS propagation... 3. TTL settings... 4. Contact your DNS provider."
- [/guides/verifying-domains](https://docs.useplunk.com/guides/verifying-domains)："Even with a verified domain, emails may go to spam if: Your content triggers spam filters... Your sender reputation is new or low. Recipients have marked your emails as spam... You're sending to invalid or unengaged contacts."
- [/concepts/campaigns](https://docs.useplunk.com/concepts/campaigns)："The from address must be on a domain you've verified for sending. Creating or updating a campaign with an unverified from returns 403."
- [/guides/verifying-domains](https://docs.useplunk.com/guides/verifying-domains)："Verification typically completes within a few minutes, but can take up to 72 hours depending on DNS propagation... If you've just added a record and don't want to wait, you can trigger a re-check from the dashboard."

## I 解释

域名验证是 Plunk 发送链路的第一道门，而且是一道**否决性**的门：`from` 落在未验证域名上时，创建或更新 campaign / template 会直接 403，不是 400 也不是 422。所以它不是"优化项"，而是"能不能发"的前置条件。

这套 DNS 记录按职责分成四块，混在一起记最容易出错。DKIM（3 条 CNAME）负责证明邮件没被篡改、防冒充；SPF（1 条 TXT）负责声明谁有权代表该域名发信；bounce MX（1 条 MX）负责把收件方产生的退信与投诉回传给 Plunk，以便自动标记问题联系人、维护发信声誉；DMARC（1 条 `_dmarc` TXT）告诉收件方前两者失败时该怎么办、报告发往哪里。前三块必做，DMARC 官方标为 optional，但因为 Gmail/Yahoo/Microsoft 对批量发件人已有普遍预期，实际应视为必做。

真正的高频失败点不是"忘了加"，而是"加错了形状"。SPF 的铁律是一个域名只能有一条 SPF TXT——已有 Google Workspace 或 M365 时必须把 Plunk 的 include **合并**进现有记录，新增第二条会让两条同时失效。MX 那两条则常被当成一条：bounce MX 处理出站反馈，inbound MX 才负责收信，两者可以只配其一；而 apex 域若已在用 Workspace 收信，把 MX 指向 Plunk 会让现有邮箱挂掉。

最后一个常被忽略的判断是：验证通过只是必要条件，不是充分条件。已经 verified 但邮件仍进垃圾箱时，根因几乎总在内容、发信声誉、历史投诉、名单质量这四类，继续改 DNS 是无效功。

因此这张卡的执行顺序固定为"先铺齐 → 再等验证 → 再排障 → 最后才谈 DMARC 收紧"，跳步（比如还没 verified 就上 `p=reject`）只会让现象更难归因。

## A1 示例

**示例 1 · 已有 SPF 时的合并演算**（来自 `/guides/verifying-domains`）

- 输入/问题：域名已在用 Google Workspace，现有记录是 `v=spf1 include:_spf.google.com ~all`，现在要接 Plunk。
- 怎么执行：不新增第二条 TXT，而是把 Plunk dashboard 给的 include 机制插入现有记录的 `v=spf1` 列表，保留原来的 `~all` 限定符。
- 产出/预期输出：`v=spf1 include:_spf.google.com include:<Plunk's include from the dashboard> ~all`。文档明确 "Two separate SPF records will cause both to fail"——若误加第二条，结果是两条同时失效、验证卡住、邮件进垃圾箱。

**示例 2 · DMARC 从 `p=none` 起步**（来自 `/guides/verifying-domains`）

- 输入/问题：团队想直接上 `p=reject` 以求"最安全"。
- 怎么执行：在 `_dmarc.yourdomain.com` 加 TXT `v=DMARC1; p=none; rua=mailto:dmarc-reports@yourdomain.com`，先只收报告、不影响投递；观察几周报告、确认所有合法发信方都通过认证后，再收紧到 `p=quarantine`，最终 `p=reject`。
- 产出/预期输出：一条可直接抄的记录模板 + 三阶段收紧路径。收紧后的投递数据文档未说明（未给出）。

**示例 3 · 记录加了 24 小时仍是 pending**（来自 `/guides/verifying-domains` 的四步排查）

- 输入/问题：三条 DKIM CNAME、SPF TXT、bounce MX 都加完了，超过 24 小时仍显示未验证。
- 怎么执行：① 核对复制的值是否精确（有无多余空格）→ ② 用 `dig` 或在线 DNS 检查工具确认记录确实已发布 → ③ 调低 TTL 排除 DNS 服务商缓存 → ④ 联系 DNS 服务商确认其对 CNAME/TXT/MX 的特殊要求或界面限制。
- 产出/预期输出：定位到具体失败环节（复制错 / 未发布 / 缓存 / 服务商限制），或直接确认已发布。

**示例 4 · 已验证仍进垃圾箱的分诊**（来自 `/guides/verifying-domains`）

- 输入/问题：域名状态显示 verified，但发出的邮件稳定进垃圾箱。
- 怎么执行：先排除 DNS 与验证问题，再依次检查四类原因——内容触发垃圾过滤（链接过多、可疑词）、发信声誉新或低、收件人此前标记过垃圾、收件人无效或不活跃。
- 产出/预期输出：归因到内容 / 声誉 / 历史投诉 / 名单质量之一，动作落到列表卫生（见 `plunk-sending-health`），而不是继续改 DNS。

## A2 何时会用

**用户场景**

1. 新域名第一次接 Plunk，需要一份能直接照抄的 DNS 记录清单。
2. 域名已经在用 Google Workspace / Microsoft 365 / 其他发信平台，担心加 SPF 会把现有发信搞坏。
3. 记录加了但一直显示 pending，或后台等不及想立刻触发一次检查。
4. 创建 campaign / template 时返回 403，或 `/v1/send` 的 `from` 被拒。
5. 域名已 verified 但邮件仍进垃圾箱，需要判断还要不要继续折腾 DNS。

**语言信号（中英双写）**

`域名验证` / `verify domain` / `DNS` / `SPF` / `DKIM` / `DMARC` / `MX record` / `MAIL FROM` / `垃圾箱` / `emails going to spam` / `deliverability` / `403` / `pending verification` / `DNS propagation`

**与相邻能力的区分**

- 本卡负责：让域名具备发信资格——DNS 记录清单、SPF 合并、验证状态排查、DMARC 上线节奏。
- `plunk-inbound-email` 负责 inbound MX 与收信后的处理（verdict、webhook），本卡只判断"要不要配 inbound MX"这一层。
- `plunk-sending-health` 负责已验证之后的退信/投诉与列表卫生，即"仍进垃圾箱"的后续动作。
- `plunk-api-essentials` 负责 403 之外的通用错误码与 requestId 排障。
- `plunk-self-hosting` 负责自托管侧的 `MAIL_FROM_SUBDOMAIN`、SES 验证与 IAM 配置。

## E 执行步骤

**输入契约**

必需信息：
1. 要验证的域名（以及是 apex 域还是子域）。
2. 该域当前已有的 DNS 现状：是否已有 SPF TXT（内容是什么）、是否已有其他发信方、apex 的 MX 现在指向谁（是否已在用 Workspace/M365 收信）。
3. DNS 控制台权限，以及服务商是谁。
4. 是否要启用收信（决定要不要配 inbound MX）。

缺失时先问：现有 SPF 记录原文、apex MX 当前指向、是否打算启用 inbound、是否自托管。

**DNS 记录清单速查**（具体值一律以 dashboard 给出的为准）

| # | 记录 | 类型 | 主机名 | 是否必做 | 作用 |
| --- | --- | --- | --- | --- | --- |
| 1 | DKIM | CNAME ×3 | Plunk 给出 | 必做 | 数字签名，防篡改与防冒充 |
| 2 | SPF | TXT ×1 | 发信域（apex 或子域） | 必做 | 授权发信服务器；**全域只能有一条** |
| 3 | bounce MX | MX ×1 | Plunk 给出 | 必做 | 接收退信与投诉反馈 |
| 4 | DMARC | TXT ×1 | `_dmarc.<domain>` | 建议 | 失败处置策略 + `rua` 聚合报告 |
| 5 | MAIL FROM 的 MX + TXT | MX ×1 + TXT ×1 | `mail.<domain>` 等子域 | 可选 | 改善 DMARC 对齐 |
| 6 | inbound MX | MX | 收信子域 | 仅启用收信时 | **接收**发往该域的邮件，与 #3 用途不同 |

**步骤**

1. **取记录值**：在 dashboard 的项目设置 Domains 页添加域名（或调用 `POST /domains`，其响应直接给出需要添加的 DNS 记录值）。完成标准：手上有一份 Plunk 生成的、逐条对应的具体值，而不是通用模板。
2. **铺必做三件套**：DKIM 3 条 CNAME + SPF 1 条 TXT + bounce MX 1 条 MX。完成标准：五条记录全部存在于 DNS 控制台，且值与 dashboard 给出的一字不差。
3. **处理 SPF 冲突**：先查现有 SPF。若已存在 → 把 Plunk 的 include 合并进现有那条 `v=spf1` 列表，保留原 `~all`/`-all` 结尾，并删除任何第二条 SPF TXT；若不存在 → 新建一条。完成标准：`dig TXT <domain>` 只返回一条 SPF 记录，且其中同时包含原有 include 与 Plunk 的 include。
4. **加 DMARC（建议）**：在 `_dmarc.yourdomain.com` 加 TXT `v=DMARC1; p=none; rua=mailto:dmarc-reports@yourdomain.com`。完成标准：能收到 rua 聚合报告。
5. **（可选）MAIL FROM 子域**：若要让 bounce 信封地址与发信域名对齐、改善 DMARC 对齐，配置 `mail.yourdomain.com` 之类的子域，并在该子域上再加 MX + TXT 各一条。完成标准：子域的两条记录已发布且 dashboard 显示通过。**冲突检查**：自托管由 `MAIL_FROM_SUBDOMAIN` 控制构造出的主机名（默认前缀 `plunk`，即 `plunk.yourdomain.com`）；若该子域已被 R2/CDN 自定义域名占用，必须改前缀，因为 MAIL FROM 主机名需要的 MX + TXT 无法与 CNAME 共存。
6. **区分两条 MX**：bounce MX 处理出站反馈（必做），inbound MX 负责收信（可选）。只在启用收信时才配 inbound MX，且该子域必须先完成发信验证，其 MX 才会被接受。完成标准：两条 MX 的用途在变更单里写清楚；若 apex 已在用 Workspace/M365 收信，inbound 一律走子域，不动 apex MX。
7. **等验证或强制检查**：后台每 5 分钟自动检查一次，通常几分钟完成，最长 72 小时（取决于 DNS 传播）。不想等可在 dashboard 手动触发重查，或调 `GET /domains/:id/verify` 强制立即检查。完成标准：Domains 页里每种记录类型均显示 verified。
8. **若 24 小时仍未验证 → 转四步排查**：① 核对复制的值（无多余空格）→ ② `dig` / 在线工具确认记录已发布 → ③ 调低 TTL 排除缓存 → ④ 联系 DNS 服务商确认其对 CNAME/TXT/MX 的特殊要求。完成标准：定位到四个环节之一；若全部通过仍 pending，记录 requestId 转支持（见 `plunk-api-essentials`）。
9. **DMARC 渐进收紧**：`p=none` 观察 → 收几周报告、确认所有合法发信方都已通过认证 → 改 `p=quarantine` → 最终 `p=reject`。完成标准：每一档收紧前都有一份"合法发信方全部通过"的报告依据；跳过观察期直接上 `p=reject` 会把未梳理干净的第三方发信方直接拒掉。
10. **验证后仍进垃圾箱 → 换赛道**：先排除 DNS 与验证问题，再按内容 / 声誉 / 历史投诉 / 名单质量依次分诊，动作落到列表卫生与内容整改。完成标准：归因到四类之一，且不再改动 DNS。
11. **⚠️ 文档口径不一致，两者并列，不要替文档做取舍**：API 参考的端点表列有 `POST /domains`（"Add a domain for verification. Returns the DNS records you need to add"）、`GET /domains/:id/verify`、`DELETE /domains/:id`；但 MCP 指南与术语表明确声明域名增删只能在 dashboard 完成（"Adding and removing domains stays in the dashboard"）。实操上以 dashboard 为主路径，不要假定一定存在可用的域名管理端点。

**症状 → 根因 → 动作速查**

| 症状 | 最可能的根因 | 动作 |
| --- | --- | --- |
| 创建 / 更新 campaign 或 template 返回 403 | `from` 域名未验证，或 DNS 校验未完成 | 回第 7 步确认 verified 再重试 |
| 记录加完 24 小时仍 pending | 值复制错 / 未发布 / TTL 缓存 / 服务商限制 | 转第 8 步四步排查 |
| `dig` 查到两条 SPF TXT | 新增了第二条而不是合并 | 转第 3 步合并，并删掉多余那条 |
| 配了 bounce MX 但仍收不到退信反馈 | 把 bounce MX 与 inbound MX 搞混 | 转第 6 步核对两条 MX 的用途 |
| 改完 apex MX 后原邮箱收不到信 | 抢占了主收信 MX | 立即改回原值，inbound 改走子域并等 TTL 过期 |
| 域名已 verified 但邮件仍进垃圾箱 | 内容 / 声誉 / 历史投诉 / 名单质量 | 转第 10 步，不要再动 DNS |
| 不想等后台的 5 分钟轮询 | 后台每 5 分钟自动检查一次 | dashboard 手动重查，或 `GET /domains/:id/verify` |
| 子域 MX 配了却不生效 | 该子域未完成发信验证 | 先让子域完成发信验证（inbound MX 只在完全验证的域名上工作） |

**输出契约**

产出：
1. 一份可直接照抄的 DNS 记录清单（类型 / 主机名 / 值 / 用途），含 SPF 合并后的单条最终值。
2. 验证状态结论（每种记录类型是否已 verified）。
3. 若排查：定位到的失败环节与下一步动作；若仍进垃圾箱：四类归因结论。
4. 一份验收后的留档（记录清单 + `dig` 结果 + dashboard 的 verified 截图），供后续 DNS 变更时对照复验。

验收：
- `dig` 结果与清单逐条一致，SPF 只有一条；
- dashboard 里 DKIM / SPF / bounce MX 三项均为 verified；
- 用该域名的 `from` 创建 campaign 不再返回 403。

## B 边界与失效

1. **不要用本卡解决"收信"问题**：inbound MX 只是启用条件之一，收信后的 verdict、路由与 webhook 见 `plunk-inbound-email`。本卡不判断邮件内容是否垃圾。
2. **不要新增第二条 SPF**：这是验证卡住最常见的根因，且症状具有迷惑性——两条记录看起来都在，但认证双双失败。
3. **不要把 bounce MX 与 inbound MX 混为一谈**：两者是两条不同的记录，用途不同，可以只配其一。误把 apex 的收信 MX 改成 Plunk 会让现有邮箱（Workspace/M365）直接中断。
4. **不要跳过 DMARC 观察期**：直接上 `p=reject` 会拒掉尚未梳理干净的第三方合法发信方。
5. **验证通过 ≠ 进收件箱**：DNS 只是前提条件。已 verified 仍进垃圾箱时根因在内容 / 声誉 / 历史投诉 / 名单质量，继续改 DNS 是无效功。
6. **MAIL FROM 与 CNAME 不能共存**：自托管默认构造 `plunk.yourdomain.com`，若该子域已被 R2/CDN 自定义域名占用会冲突，必须改 `MAIL_FROM_SUBDOMAIN` 前缀。
7. **高危不可撤销操作**：删除已验证域名（dashboard 或 `DELETE /domains/:id`）会让该域名上的 `from` 立即失效，并连带影响引用它的 template 与 campaign；删除前先确认没有在跑的活动或 workflow 依赖它。DNS 侧把 apex MX 改指向 Plunk 同样属于高影响变更——会中断既有收信，不可通过 Plunk 侧撤销，只能改回原值并等 TTL 过期。
8. **403 不一定是 DNS 问题**：`PROJECT_DISABLED`、`EMAIL_VERIFICATION_REQUIRED` 也会返回 403（见 `plunk-api-essentials`）。确认域名 verified 后仍 403，应转向项目状态排查而不是继续查 DNS。
9. **验证时间窗有上下界**：几分钟到 72 小时。不到 24 小时就开始四步排查通常过早；超过 72 小时仍 pending 则应视为异常而非继续等待。
10. **不要以为验证一次就永远有效**：DNS 服务商变更、TTL 大幅调整、新增第三方发信方（尤其是又加了一条 SPF 的那个）都可能让已通过的记录回退。每次 DNS 侧有变更都应回到第 7 步复验。
11. **MAIL FROM 不是所有收件方的硬性要求**：文档的表述是"required by some inbox providers for full pass-through"（部分收件箱要求）。是否启用取决于目标收件箱与 DMARC 对齐需求，不要无脑配置——它会在子域上多出 MX + TXT 两条需要维护的记录。
12. **本卡不覆盖自托管的 SES 侧配置**：自托管时 Plunk 会自动向 AWS SES 处理验证与 DKIM，但 IAM 策略、`MAIL_FROM_SUBDOMAIN` 前缀冲突等属于部署侧，见 `plunk-self-hosting`。
