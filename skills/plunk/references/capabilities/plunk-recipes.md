# 端到端配方

## R 原文依据

- [/recipes/waitlist](https://docs.useplunk.com/recipes/waitlist)："A waitlist is the simplest possible Plunk workflow: one tracked event from your app, one workflow that listens for it, one email... A workflow's trigger event cannot be changed after the first execution. Namespace it (`waitlist.joined`) rather than something generic you might want to reuse."
- [/recipes/waitlist](https://docs.useplunk.com/recipes/waitlist)："If you want to segment on waitlist signups later, add an UPDATE_CONTACT step before the email: `{ "stage": "waitlist", "waitlistSource": "{{event.referrer}}" }`... This is cleaner than filtering on 'ever fired waitlist.joined'."
- [/recipes/double-opt-in](https://docs.useplunk.com/recipes/double-opt-in)："The trick is `{{subscribeUrl}}`: a per-contact link Plunk auto-injects into every send. Clicking it flips `subscribed` to `true` and fires a `contact.subscribed` event."
- [/recipes/double-opt-in](https://docs.useplunk.com/recipes/double-opt-in)："A marketing template targeted at an unsubscribed contact is silently skipped. Use a transactional template for the confirmation specifically — it bypasses the subscription check."
- [/recipes/double-opt-in](https://docs.useplunk.com/recipes/double-opt-in)："Both calls pass `subscribed: false`. If you skip the first call and rely on `/v1/track` alone, tracking on an unknown email creates the contact — but defaults it to subscribed, which defeats the point."
- [/recipes/double-opt-in](https://docs.useplunk.com/recipes/double-opt-in)："WAIT_FOR_EVENT: Event `contact.subscribed`, Timeout `86400` (24 hours). On timeout, send a single reminder (also transactional). Keep the number of reminders small — repeated confirmation prompts look like spam."
- [/recipes/sync-unsubscribes](https://docs.useplunk.com/recipes/sync-unsubscribes)："That flip also fires `contact.unsubscribed`, meaning your own webhook will round-trip back into your handler. That's usually harmless because the update is idempotent — but be aware of it."
- [/recipes/sync-unsubscribes](https://docs.useplunk.com/recipes/sync-unsubscribes)："Build a second workflow with the same shape, triggered by `contact.subscribed`. Keep it separate from the unsubscribe flow — two short workflows are easier to monitor than one branched one."

## I 解释

配方层解决的是"我已经知道概念，但现在要拼出一个能跑的东西"——它把联系人、事件、模板、workflow、webhook 这几个零件按固定顺序组装起来，并给出每个接缝处的默认取值。三个官方配方（waitlist、double opt-in、sync unsubscribes）看似不同，实则共享同一套判断：**订阅状态是硬约束、事件名不可逆、状态要物化到联系人上、流程要短而可观测**。

第一个决策是**事件命名**。workflow 的触发事件在**首次执行之后不可更改**，所以事件名必须带命名空间（`waitlist.joined`），不能取一个以后想复用的泛化名字。这条约束决定了命名是"一次性决策"，配方的第一步就该锁定它，而不是等 workflow 跑起来再改。

第二个决策是**订阅状态的建立方式**。`/v1/track` 打到未知邮箱会自动建联系人，且默认 `subscribed` 为 true；所以任何需要"先未订阅、后确认"的场景（double opt-in）都必须两次调用都显式传 `subscribed: false`——先 `POST /contacts` 建人，再 `/v1/track` 打事件。省掉第一次调用等于绕过 opt-in。

第三个决策是**模板类型**。确认邮件的收件人此刻必然是未订阅状态，用 MARKETING 模板会被**静默跳过**（不发也不报错），这是整条链路最难排查的失败；所以确认信与催确认提醒都必须用 TRANSACTIONAL，只有确认后的欢迎邮件才用 MARKETING。三者通过 `{{subscribeUrl}}` 串起来：点击即置 `subscribed=true` 并触发 `contact.subscribed`，后者又可以作为欢迎流程的触发器，同时覆盖首次确认与重新订阅。

第四个决策是**状态物化而非事件存在性**。想在后续分群里圈出"等待列表报名者"，应在 workflow 里加 `UPDATE_CONTACT` 步骤把 `stage: waitlist` 写进 `contact.data`，再按字段建 segment；这比过滤"是否触发过某事件"更快、语义更明确，也能与其它条件自由组合。

第五个决策是**编排粒度**。双向同步场景里，订阅与退订各建一个短 workflow，而不是合成一个带分支的长 workflow——可观测性优先于图的最简性。并且必须知道：当你的产品是事实源、用 `PATCH /contacts/:id` 反向改状态时，这次翻转同样会触发 `contact.unsubscribed`，于是回环进你自己的 webhook handler；因为更新是幂等的通常无害，但回环是设计的一部分，不是 bug。

第六个决策是**接收端的形状**。同步链路的入口是一个公开 HTTPS 端点，它必须做三件事：校验共享密钥（不匹配直接 401）、按 `contact.email` 幂等更新（而不是按 id，因为 Plunk 侧给你的邮箱）、在 **10 秒内**返回 2xx——webhook 请求 10 秒超时，所以慢活要异步化。`event.reason` 是区分来源的关键：`bounce` / `complaint` 表示系统自动退订，缺省则视为 `user_action`。把这三类分开统计，否则会把"地址质量差"误读成"内容不受欢迎"。

## A1 示例

**示例 1 · Waitlist：一次 track 驱动一封确认邮件**（来自 `/recipes/waitlist` 的 worked example）

- 输入/问题：报名表单提交后要自动发一封确认邮件。
- 怎么执行：① 建一个 MARKETING 模板，用 `{{firstName}}` / `{{product}}` 占位；② 后端用 `sk_` 调 `POST /v1/track`，body `{ "event": "waitlist.joined", "email": "ada@example.com", "data": { "firstName": "Ada", "product": "Beta" } }`——**绝不能从浏览器调**；③ 建 workflow（触发 `EVENT` 在 `waitlist.joined` + `SEND_EMAIL`）并 enable。
- 产出/预期输出：track 会 upsert 联系人（默认订阅）并在其上记录 `waitlist.joined`，`data` 落到联系人上可在模板里直接引用；workflow 启用后每个加入者收到一封确认邮件。默认 disabled，不开开关不触发。文档未给出投递/转化数据。

**示例 2 · Waitlist 打标：用 UPDATE_CONTACT 替代事件存在性过滤**（来自 `/recipes/waitlist` 的 Tagging 小节）

- 输入/问题：以后要针对等待列表人群做二次触达，怎么圈人最干净。
- 怎么执行：在 SEND_EMAIL 之前加一个 UPDATE_CONTACT 步骤，写入 `{ "stage": "waitlist", "waitlistSource": "{{event.referrer}}" }`（事件上下文可在步骤里直接引用）。
- 产出/预期输出：之后用 `stage == "waitlist"` 建 segment 定向即可；文档称这比过滤"曾触发过 `waitlist.joined`"更干净，未给出性能数据。

**示例 3 · Double opt-in：两个 workflow 用 `{{subscribeUrl}}` 串起确认与欢迎**（来自 `/recipes/double-opt-in` 的 Setup）

- 输入/问题：注册后要先确认再收营销邮件。
- 怎么执行：① 建两个模板——TRANSACTIONAL 确认信（含 `<a href="{{subscribeUrl}}">` 链接）与 MARKETING 欢迎信；② 后端用 `sk_` 先建人再打事件，**两次都显式 `subscribed: false`**：

  ```bash
  curl https://next-api.useplunk.com/contacts \
    -H "Authorization: Bearer sk_your_secret_key" \
    -d '{ "email": "ada@example.com", "subscribed": false, "data": { "firstName": "Ada" } }'

  curl https://next-api.useplunk.com/v1/track \
    -H "Authorization: Bearer sk_your_secret_key" \
    -d '{ "event": "signup.pending", "email": "ada@example.com", "subscribed": false }'
  ```

  ③ Workflow A：触发 `EVENT` 在 `signup.pending` → 发确认信；④ Workflow B：触发 `EVENT` 在 `contact.subscribed` → 发欢迎信；两条都 enable。
- 产出/预期输出：点击 `{{subscribeUrl}}` 把 `subscribed` 置 true 并触发 `contact.subscribed`，从而触发欢迎流程。Workflow B 复用 `contact.subscribed`，可同时覆盖首次确认与重新订阅。文档未给出确认率数据。

**示例 4 · 未确认就催一次**（来自 `/recipes/double-opt-in` 的 Reminder 小节）

- 输入/问题：有人提交了表单但一直没点确认链接。
- 怎么执行：在 Workflow A 的发送步骤后加 `WAIT_FOR_EVENT`：Event `contact.subscribed`、Timeout `86400`（秒，即 24 小时）；超时分支发一封提醒，提醒也必须是 TRANSACTIONAL。
- 产出/预期输出：24 小时未确认者收到一次提醒。提醒次数要少——反复催确认对邮箱服务商和收件人都像垃圾邮件。文档未给出"几次算多"的取值。

**示例 5 · 退订双向同步与回环**（来自 `/recipes/sync-unsubscribes` 的 worked example）

- 输入/问题：要让自家 user 表与 Plunk 的订阅状态保持一致。
- 怎么执行：① 建一个公开 HTTPS 端点，校验 `Authorization: Bearer <共享密钥>`，失败返 401，成功后按 `contact.email` 更新 `emailSubscribed` 与 `emailUnsubscribedReason: event.reason ?? 'user_action'` 并返 204（webhook 10 秒超时，慢活要异步化）：

  ```ts
  app.post('/plunk/unsubscribes', async (req, res) => {
    if (req.header('authorization') !== `Bearer ${process.env.PLUNK_WEBHOOK_SECRET}`) {
      return res.status(401).end();
    }
    const { contact, event } = req.body;
    await db.user.update({
      where: { email: contact.email },
      data: {
        emailSubscribed: false,
        emailUnsubscribedReason: event.reason ?? 'user_action',
      },
    });
    res.status(204).end();
  });
  ```

  ② Workflow 1：触发 `contact.unsubscribed` → `WEBHOOK` 步骤（填 URL + Bearer 头，body 留空用默认 payload）；③ Workflow 2：同样形状，触发 `contact.subscribed`，不与退订流程合并；④ 反向：`PATCH /contacts/:id` body `{"subscribed": false}`。
- 产出/预期输出：`event.reason` 为 `bounce` / `complaint` 时是自动退订，人工或自助退订时缺省。反向的 PATCH 也会触发 `contact.unsubscribed` 并回环进你自己的 handler；文档说明因更新幂等通常无害，但提醒必须意识到这一点。

## A2 何时会用

**用户场景**

1. 刚接完 Plunk，想照着一个能跑通的最小闭环（报名即发信）验证链路。
2. 要做注册确认 / 双重 opt-in，担心合规与"邮件发出去但没人收到"。
3. 要把 Plunk 的订阅状态回写到自家数据库，或让自家设置页成为事实源。
4. 报名后要做二次触达，纠结"按事件圈人还是按字段圈人"。
5. 已有 workflow 越写越长、越难排障，想知道该拆还是该加分支。

**语言信号**

`waitlist` / `等待列表` / `double opt-in` / `双重确认` / `确认邮件` / `confirmation email` / `subscribeUrl` / `signup.pending` / `contact.subscribed` / `contact.unsubscribed` / `退订同步` / `sync unsubscribes` / `webhook` / `回写` / `UPDATE_CONTACT` / `打标` / `WAIT_FOR_EVENT` / `超时提醒` / `recipe` / `配方` / `端到端`

**与相邻能力的区分**

- 本卡负责：**跨多个能力的组装顺序与接缝取值**（事件名、模板类型、状态字段、流程拆分粒度）。
- `plunk-workflows` 负责 workflow 自身的概念、步骤类型与触发语义；本卡只给出"这样编排是对的"这一类组合结论。
- `plunk-subscription-consent` 负责订阅状态与合规的口径（退订途径、snooze、归因）；本卡只消费"marketing 模板跳过未订阅联系人"这一条硬约束。
- `plunk-tracking-events` 负责 `/v1/track` 的字段语义与保留事件名；本卡只消费"track 会 upsert 联系人且默认订阅"这一点。
- `plunk-webhooks` 负责 WEBHOOK 步骤的 payload 与重试语义；本卡只给出接收端的最小写法与回环认知。
- `plunk-templates-liquid` 负责模板类型与变量渲染；本卡只决定"这一封该用哪种类型"。

## E 执行步骤

**输入契约（必需信息）**

- 要实现的场景：waitlist / double opt-in / 双向同步中的哪一种。
- 谁发起事件（前端还是后端）、用哪个 key（`sk_` 还是 `pk_`）、事件名是否已定。
- 自家系统是否是订阅状态的事实源；若是，接收端点的 URL 与共享密钥。
- 缺失时先问：场景、发信域名是否已验证（未验证则先转 `plunk-domain-deliverability`）、端点是否可达。

**输出契约（产出与验收）**

- 一份按顺序排列的调用清单（API 调用 + 模板 + workflow 步骤）+ 接收端代码骨架。
- 验收：发一次真实事件后，workflow 的 execution 出现且邮件到达；double opt-in 场景额外要求确认前 `subscribed` 为 false、点击后变为 true。

**步骤**

1. **锁定事件名**：带命名空间（`waitlist.joined` / `signup.pending`），不用泛化名。完成标准：事件名写死在代码里，且确认它在首次执行后不可更改。
2. **确认发信域已验证**：模板的 `from` 必须在已验证域名上，否则 403。完成标准：域在 dashboard 显示已验证（未验证转 `plunk-domain-deliverability`）。
3. **建模板并选对类型**：waitlist 用 MARKETING；double opt-in 的确认信与催确认提醒**必须** TRANSACTIONAL，欢迎信才用 MARKETING，确认信里放 `{{subscribeUrl}}`。完成标准：模板类型与"收件人此刻是否已订阅"匹配——收件人必然未订阅的一律 TRANSACTIONAL。
4. **建联系人 / 打事件**：waitlist——后端用 `sk_` 调 `POST /v1/track`（绝不从浏览器）；double opt-in——先 `POST /contacts` 带 `subscribed: false`，再 `/v1/track` 同样带 `subscribed: false`。完成标准：double opt-in 场景下确认前该联系人 `subscribed` 为 false；若发现自动被建成已订阅，说明漏了显式传值（转第 4 步重做）。
5. **建 workflow 并 enable**：waitlist——`EVENT` 在 `waitlist.joined` → （可选 `UPDATE_CONTACT` 打标）→ `SEND_EMAIL`；double opt-in——Workflow A `EVENT` 在 `signup.pending` → `SEND_EMAIL`（确认信），Workflow B `EVENT` 在 `contact.subscribed` → `SEND_EMAIL`（欢迎信）。完成标准：workflow 的 enabled 开关已打开（默认是 disabled，不开不触发）。
6. **（按需）加催确认分支**：Workflow A 的发送步骤后加 `WAIT_FOR_EVENT`（`contact.subscribed`，Timeout `86400`），超时发一封 TRANSACTIONAL 提醒。完成标准：提醒次数保持在很小的量级，且提醒模板也是 TRANSACTIONAL。
7. **（按需）双向同步**：建接收端点（验 Bearer 共享密钥、按 email 幂等更新、10 秒内返回 2xx）；建两条**独立**的短 workflow——`contact.unsubscribed` → WEBHOOK 与 `contact.subscribed` → WEBHOOK，body 留空用默认 payload。完成标准：两条流程分别可观测；不要合并成一条带分支的流程。
8. **（按需）反向同步并接受回环**：产品是事实源时用 `PATCH /contacts/:id` 改 `subscribed`。完成标准：明确知道这次翻转也会触发 `contact.unsubscribed` 并回环进自己的 handler，靠 handler 幂等吸收；若 handler 不幂等，先在 handler 侧做来源去重再上线。

## B 边界与失效

1. **不要用 MARKETING 模板发确认邮件**。收件人此刻正是未订阅状态，会被静默跳过——不发也不报错，确认链路直接死循环，后续欢迎流程也不触发。
2. **不要只调 `/v1/track` 就当完成了双重 opt-in**。未知邮箱会被建成 `subscribed` 默认 true，等于绕过 opt-in，带来合规风险与垃圾投诉上升。
3. **不要在 workflow 首跑之后才发现事件名取错了**。触发事件名不可更改，只能重建 workflow；命名时就要带命名空间。
4. **不要把订阅与退订合并成一条带分支的长 workflow**。两个短 workflow 更容易监控与排障，可观测性优先于图的最简性。
5. **不要忽略反向同步的回环**。`PATCH /contacts/:id` 会触发 `contact.unsubscribed` 并回到你自己的 handler；handler 不幂等就会放大成重复写入或循环。
6. **不要让催确认变成骚扰**。提醒次数要少，且提醒同样是 TRANSACTIONAL 模板——反复确认提示对邮箱服务商和收件人都像垃圾邮件。
7. **不要在浏览器里调 `/v1/track`**。必须用 `sk_`，从后端发起；泄露 sk_ 等于交出项目级写权限（见 `plunk-api-essentials`）。
8. **文案/设计类问题不在本卡范围**。配方只解决"怎么接起来"，不解决邮件内容质量、发送时机策略与统计口径（见 `plunk-campaigns-analytics`）。
9. **文档缺口（能力边界）**：配方的 workflow 步骤都以 dashboard 操作描述，Workflow API 只有路由表没有完整请求 schema（见 `plunk-workflows`），无法保证用纯 API 复刻同样的步骤结构；`WAIT_FOR_EVENT` 的超时单位在文档中以 86400 这一取值出现，未给出单位之外的边界说明。
10. **高危/难撤销操作**：workflow 一旦首次执行，触发事件名永久锁定；把已入库联系人的 `subscribed` 批量置为 true 会绕过此前建立的 opt-in 证据链，不可撤销。
11. **不要让接收端点做慢活**。webhook 10 秒超时，超时的请求不会被当作成功；把重活丢进队列，端点本身尽快返 2xx。
12. **不要把 `event.reason` 缺省当成"未知错误"**。缺省就是人工或自助退订，应归为 `user_action`；只有 `bounce` / `complaint` 才是系统自动退订（见 `plunk-subscription-consent`）。
