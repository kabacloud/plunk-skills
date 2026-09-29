# MCP 与 AI agent 集成

## R 原文依据

- [/guides/mcp-server](https://docs.useplunk.com/guides/mcp-server)："The official Plunk MCP server lets an AI agent work in your Plunk project: sending transactional email, managing contacts and segments, and drafting and sending campaigns. It works with both the hosted product and self-hosted instances."
- [/guides/mcp-server](https://docs.useplunk.com/guides/mcp-server)："You need a **secret key** (`sk_…`) from **Settings → API Keys**."
- [/guides/mcp-server](https://docs.useplunk.com/guides/mcp-server)："Sends ask for confirmation... Confirmation is not something the model can supply itself — it comes from you, through your MCP client. A client that cannot show a prompt cannot send."
- [/guides/mcp-server](https://docs.useplunk.com/guides/mcp-server)："Read-only mode is structural. With `PLUNK_READ_ONLY=true` the mutating tools are never registered, so they cannot be invoked even by name."
- [/guides/mcp-server](https://docs.useplunk.com/guides/mcp-server)："Account-level actions are out of reach. Billing, project deletion, and key rotation all require a dashboard session rather than an API key, so no tool can reach them."
- [/guides/mcp-server](https://docs.useplunk.com/guides/mcp-server)："If you self-host, the Plunk API server itself uses a variable called PLUNK_API_KEY for its platform notification emails — and that key belongs to a different project. When both live in the same environment, set `PLUNK_MCP_API_KEY` so the agent works on the project you intend."
- [/guides/mcp-server](https://docs.useplunk.com/guides/mcp-server)："`plunk_send_email` is for one-off messages to specific people. Reaching a whole list means creating a campaign and then sending it. The agent is instructed not to loop sends over an audience, which would hit the rate limit and skip unsubscribe handling."
- [/guides/mcp-server](https://docs.useplunk.com/guides/mcp-server)："`plunk_test_campaign` sends one copy for review. The recipient must be a member of the Plunk project; any other address is rejected... `plunk_subscribe_contact` reverses it, but only for a contact that already exists — opting someone in never creates a new record."

## I 解释

MCP server（npm 包 `@plunk/mcp`）把 Plunk 的项目能力暴露成一组工具，让 agent 能在项目里直接干活。它解决的核心问题不是"能不能调 API"，而是**`sk_` 是项目级全权凭据，交给一个模型之前必须先把它关进笼子**——所以这一卡的主体不是工具清单，而是那三层安全阀与若干刻意不做的事。

第一层是**发送必须人工确认**。发 campaign、或给多于一个收件人发信时，会先提示并告知收件人数。确认**不能由模型自己给出**，它来自人、经由 MCP 客户端；因此"客户端能不能弹提示"是使用前的**前置前提**——不能提示的客户端根本发不出去，headless / CI 场景下只能显式设 `PLUNK_ALLOW_UNCONFIRMED_SENDS=true` 跳过（文档限定为 headless automation only）。

第二层是**只读是结构性的**。`PLUNK_READ_ONLY=true` 时写工具**根本不注册**，按名字也调不动——这不是运行时拦截，而是工具集本身被裁剪，所以不存在"绕过只读"的调用路径。

第三层是**账户级动作够不着**。计费、删项目、轮换密钥都需要 dashboard 会话而非 API key，因此没有任何工具能触及。域名的增删同理被刻意留在 dashboard：那两个端点跳过 admin 角色校验，暴露出来会让 agent 获得非 admin 成员本不该有的权限。

第四层是**agent 的操作约定**，这些是被写进 server 指令里的：群发必须走 create campaign + send campaign，不许对受众循环 `plunk_send_email`（会撞限流且跳过退订处理）；发送前用 `plunk_list_domains` 检查发信域已验证，`plunk_check_domain` 在加记录后重跑 DNS 检查；测试发送前先 `plunk_test_campaign`，收件人必须是项目成员；退订按精确 + 大小写不敏感匹配，不做子串搜索；`plunk_subscribe_contact` 只对已存在的联系人生效，永远不会新建档案。

第五层是**自托管的命名冲突**。Plunk 服务端自己也用 `PLUNK_API_KEY`（用来发平台通知邮件），而这个 key 属于**另一个项目**。两者同居一个环境时必须用 `PLUNK_MCP_API_KEY`（以及 `PLUNK_MCP_API_URL`）——它们优先于不带 `MCP_` 前缀的同名变量。这是自托管接 MCP 时最容易踩、又最难察觉的一类错误：不报错，只是 agent 一直在操作错的项目。

第六层是**环境变量的取舍**。只有 `PLUNK_API_KEY` 是必填（且必须是 `sk_`）；`PLUNK_PUBLIC_KEY` 是纯优化项——配上它事件追踪从两次请求降为一次，不配时 server 会用 `sk_` 先建/更新联系人再记事件，**两种途径联系人都会被创建**，所以"没配 pk_ 就不会建人"是错误预期。`PLUNK_READ_ONLY` 与 `PLUNK_ALLOW_UNCONFIRMED_SENDS` 是两个方向相反的信任开关：前者收紧到只剩十个只读工具，后者放开最危险的一步（跳过发送前确认），文档明确把它限定在 headless automation 场景。

## A1 示例

**示例 1 · 三种接入配置**（来自 `/guides/mcp-server` 的 Setup）

- 输入/问题：要把 Plunk MCP 接进 Claude Code / Claude Desktop / Cursor，或在自托管上接。
- 怎么执行：Claude Code 用 `claude mcp add plunk --env PLUNK_API_KEY=sk_your_key -- npx -y @plunk/mcp`；Desktop/Cursor 在 `mcpServers` 里写 `{ "command": "npx", "args": ["-y", "@plunk/mcp"], "env": { "PLUNK_API_KEY": "sk_your_key" } }`；自托管再加一项 `"PLUNK_API_URL": "https://api.your-domain.com"`。
- 产出/预期输出：文档未说明验证方式；可确定的部分是凭据必须是 `sk_`（来自 Settings → API Keys），自托管必须指向自己的 API 域。若与 Plunk 服务端共用环境，改写成 `PLUNK_MCP_API_KEY` / `PLUNK_MCP_API_URL`。

**示例 2 · 自托管的变量名撞车**（来自 `/guides/mcp-server` 的 warn Callout）

- 输入/问题：同一台机器上既跑 Plunk 服务端又跑 MCP server，agent 明明配了正确的 `sk_`，改的却是别的项目的数据。
- 怎么执行：识别到 `PLUNK_API_KEY` 在服务端的语义是"平台通知邮件用的 key"（属于另一个项目），与 MCP 侧的语义不同；改用 `PLUNK_MCP_API_KEY`（以及 `PLUNK_MCP_API_URL`），它们优先于同名变量。
- 产出/预期输出：agent 操作回到预期的项目。文档未说明冲突时的报错形态，因此这类问题表现为"静默改错项目"而非报错。

**示例 3 · headless 环境里发送永远不执行**（来自 `/guides/mcp-server` 的 Safety 与 Things to know）

- 输入/问题：在 CI / 无人值守脚本里让 agent 发 campaign，流程一直卡在确认环节。
- 怎么执行：理解"确认来自人、经由 MCP 客户端"——模型无法自给；确认客户端类型，无交互能力的客户端只能设 `PLUNK_ALLOW_UNCONFIRMED_SENDS=true`（文档限定 headless automation only）。
- 产出/预期输出：跳过提示后发送可执行。同时注意：即使可发，也不要让 agent 对受众循环逐个 `plunk_send_email`，应改为 create campaign 再 send campaign——循环发送会撞限流且跳过退订处理。

**示例 4 · 测试发送与订阅补登的两个"静默无效"**（来自 `/guides/mcp-server` 的 Things to know）

- 输入/问题：让 agent 给一个外部邮箱发测试邮件；或让 agent "把 XX 加进订阅名单"。
- 怎么执行：测试收件人必须换成项目成员地址（其它地址被拒）；补登订阅前先确认联系人已存在，不存在则先 create contact。
- 产出/预期输出：测试发出一封供审阅的副本；`plunk_subscribe_contact` 只对已存在联系人生效，opt-in 永不新建记录——若联系人不存在则什么都不发生（静默无效果）。取消用 `plunk_cancel_campaign`，它只能拦住尚未发出的部分，无法召回已发出的。

**示例 5 · 只读模式下的工具集裁剪**（合成演练，按 `/guides/mcp-server` 的工具清单可复算）

- 输入/问题：想让 agent 只读地查联系人、看活动统计、检查域名状态，不希望它有任何写入可能。
- 怎么执行：设 `PLUNK_READ_ONLY=true`；此时只有 `plunk_list_contacts`、`plunk_get_contact`、`plunk_verify_email`、`plunk_list_templates`、`plunk_list_campaigns`、`plunk_get_campaign`、`plunk_get_campaign_stats`、`plunk_list_segments`、`plunk_list_domains`、`plunk_check_domain` 被注册。
- 产出/预期输出：写类工具（create/update/subscribe/unsubscribe/delete contact、`plunk_send_email`、`plunk_track_event`、`plunk_create_template`、`plunk_create_campaign`、`plunk_test_campaign`、`plunk_send_campaign`、`plunk_cancel_campaign`、`plunk_create_segment`）根本不在工具列表里，按名字调用也无效。

## A2 何时会用

**用户场景**

1. 想让 Claude / Cursor 之类的 agent 直接在 Plunk 项目里查联系人、起草并发送邮件或活动。
2. 要给 agent 授权但担心它乱发邮件、乱改联系人——需要知道有哪些结构性护栏。
3. 自托管实例上接 MCP，agent 行为异常或操作到了错误的项目。
4. 在 CI / 无人值守流程里用 MCP 做自动发送，一直卡住发不出去。
5. 让 agent 做测试发送、补登订阅、增删域名，结果静默无效果或被拒。

**语言信号**

`MCP` / `AI agent` / `Claude` / `Cursor` / `@plunk/mcp` / `mcpServers` / `PLUNK_API_KEY` / `PLUNK_MCP_API_KEY` / `PLUNK_READ_ONLY` / `只读` / `read-only` / `确认提示` / `confirmation prompt` / `PLUNK_ALLOW_UNCONFIRMED_SENDS` / `headless` / `sk_` / `密钥轮换` / `rotate keys` / `plunk_test_campaign` / `plunk_send_campaign` / `plunk_list_domains` / `test recipient`

**与相邻能力的区分**

- 本卡负责：**MCP server 的接入配置、工具集边界、安全阀与 agent 操作约定**。
- `plunk-api-essentials` 负责 `pk_`/`sk_` 的语义、错误信封与限流响应；本卡只消费"必须用 `sk_`"这一条。
- `plunk-domain-deliverability` 负责 DNS 记录本身与验证流程；本卡只处理 agent 侧的 `plunk_list_domains` / `plunk_check_domain` 预检，**域名的增删不在 MCP 内**。
- `plunk-campaigns-analytics` 负责 campaign 的状态机与统计口径；本卡只给出"群发必须走 campaign、不许循环 send"这一条 agent 约束。
- `plunk-subscription-consent` 负责订阅状态与合规；本卡只处理 `plunk_subscribe_contact` / `plunk_unsubscribe_contact` 的匹配语义与"不新建档案"边界。
- `plunk-self-hosting` 负责自托管部署本身；本卡只负责自托管下的 `PLUNK_MCP_API_KEY` 命名冲突。

## E 执行步骤

**输入契约（必需信息）**

- 宿主客户端是什么（Claude Code / Claude Desktop / Cursor / 自研），**能否展示交互式确认提示**。
- 是托管还是自托管；自托管时 API 域名是什么，MCP 是否与 Plunk 服务端共用同一个环境。
- 期望 agent 做什么：只读查询、一对一事务发送、还是建活动群发。
- 缺失时先问：客户端能否弹提示（不能则整个发送能力不可用）、项目 secret key（`sk_`）来源、自托管 API 域。

**输出契约（产出与验收）**

- 一份可直接粘贴的 MCP 配置（含 env），加一份"哪些事 agent 做不到"的边界说明。
- 验收：agent 能列出只读工具；只读模式下写工具不在工具列表内；需要发送时客户端确实收到确认提示。

**环境变量表**（`/guides/mcp-server`）

| 变量 | 必填 | 说明 |
| --- | --- | --- |
| `PLUNK_API_KEY` | Yes | 项目的 secret key（`sk_…`） |
| `PLUNK_API_URL` | No | API base URL，自托管时指向自己的 API 域 |
| `PLUNK_PUBLIC_KEY` | No | 配了让事件追踪从两次请求降为一次 |
| `PLUNK_READ_ONLY` | No | `true` 只注册只读工具 |
| `PLUNK_ALLOW_UNCONFIRMED_SENDS` | No | `true` 跳过发送前确认，仅 headless automation |
| `PLUNK_MCP_API_KEY` | No | 优先于 `PLUNK_API_KEY`，用于同名变量已被占用的环境 |
| `PLUNK_MCP_API_URL` | No | 优先于 `PLUNK_API_URL`，同因 |

**步骤**

1. **先确认宿主能力（前置前提）**：⚠️ 文档口径：发送必须经人类确认，**无法展示确认提示的客户端根本发不出邮件**。完成标准：明确回答"客户端能否弹提示"；不能则发送类需求改走 `PLUNK_ALLOW_UNCONFIRMED_SENDS=true`（仅限 headless automation）或改走 API（见 `plunk-transactional-send`）。
2. **取 secret key**：从 Settings → API Keys 拿 `sk_…`。完成标准：配置里用的是 `sk_`；`pk_` 不够。
3. **写接入配置**：Claude Code 用 `claude mcp add plunk --env PLUNK_API_KEY=sk_your_key -- npx -y @plunk/mcp`；其它客户端在 `mcpServers.plunk` 里写 `command: npx`、`args: ["-y", "@plunk/mcp"]`、`env.PLUNK_API_KEY`：

   ```json
   {
     "mcpServers": {
       "plunk": {
         "command": "npx",
         "args": ["-y", "@plunk/mcp"],
         "env": {
           "PLUNK_API_KEY": "sk_your_key",
           "PLUNK_API_URL": "https://api.your-domain.com"
         }
       }
     }
   }
   ```

   完成标准：client 能列出 Plunk 工具集。
4. **自托管补两件事**：加 `PLUNK_API_URL`（或 `PLUNK_MCP_API_URL`）指向自己的 API 域；若与 Plunk 服务端共用环境，**必须**改用 `PLUNK_MCP_API_KEY` / `PLUNK_MCP_API_URL`。完成标准：agent 读到的项目是你预期的那个；若发现操作到了别的项目的（或改不动正确项目的）数据，回到这一步查同名变量覆盖。
5. **按信任级别选只读模式**：只做查询/审查时设 `PLUNK_READ_ONLY=true`。完成标准：工具列表里只剩十个只读工具，写工具根本不注册、按名字也调不动。
6. **（可选）配 public key**：设 `PLUNK_PUBLIC_KEY`（`pk_…`）可让事件追踪从两次请求降为一次。完成标准：`plunk_track_event` 走单请求路径；不配时 server 会用 `sk_` 先建/更新联系人再记事件，两种途径联系人都会建。
7. **发送前预检域名**：让 agent 先 `plunk_list_domains` 确认发信域已验证（未验证的发送会被拒）；刚加完记录用 `plunk_check_domain` 重跑 DNS 检查。完成标准：发信地址落在已验证域名上；**域名本身的增删只能去 dashboard**。
8. **区分一对一与群发**：一对一用 `plunk_send_email`；触达整个列表必须 `plunk_create_campaign` 再 `plunk_send_campaign`。完成标准：不存在"对受众循环 send_email"的调用序列（那会撞限流且跳过退订处理）；发出后如需中断用 `plunk_cancel_campaign`，它只能拦住尚未发出的部分。
9. **测试发送走项目成员**：`plunk_test_campaign` 的收件人必须是项目成员，其它地址被拒。完成标准：测试副本送达一个项目成员地址。
10. **订阅/退订按语义使用**：`plunk_unsubscribe_contact` 直接给邮箱，精确且大小写不敏感匹配；`plunk_subscribe_contact` 只对已存在的联系人生效。完成标准：补登订阅前先确认联系人已存在（不存在则先 `plunk_create_contact`），否则指令静默无效果。
11. **隔离与轮换**：不想让 agent 碰的资源放进独立项目；机器失信任就轮换密钥。完成标准：轮换时知道**一次会替换两个 key**；计费、删项目、轮换密钥本身都做不到，必须去 dashboard。

## B 边界与失效

1. **不要在无法交互的客户端里指望 agent 自动发信**。确认来自人、经由 MCP 客户端，不能提示就不能发送——这是宿主前提，不是配置问题。
2. **不要以为设了只读就只是"软限制"**。只读是结构性的：写工具根本不注册，按名字也调不动；反过来也不要期待只读模式下任何写操作能成功。
3. **不要让 agent 循环 `plunk_send_email` 群发**。会撞限流且跳过退订处理；群发必须建 campaign 再发送。
4. **不要让 agent 给非项目成员发测试邮件**。`plunk_test_campaign` 的收件人必须是项目成员，其它地址一律被拒。
5. **不要让 agent "把人加进订阅名单"而不先建联系人**。`plunk_subscribe_contact` 只对已存在的联系人生效，opt-in 永不新建记录——不存在联系人时什么都不发生。
6. **自托管不要沿用 `PLUNK_API_KEY`**。服务端用它发平台通知邮件、属于另一个项目，同名变量共存会让 agent 静默操作错的项目；改用 `PLUNK_MCP_API_KEY` / `PLUNK_MCP_API_URL`。
7. **域名增删不在 MCP 内**。那两个端点跳过 admin 角色校验，暴露会给 agent 非 admin 成员本不该有的权限——加域、删域只能走 dashboard（见 `plunk-domain-deliverability`）。
8. **账户级动作一律够不着**：计费、删项目、轮换密钥需要 dashboard 会话，没有工具能触达；不要向 agent 提这类需求。
9. **不要在不可信机器上长期持有 `sk_`**。它是项目级全权凭据；轮换会一次性替换两个 key，且轮换本身 agent 做不到。
10. **文档缺口（能力边界）**：文档未给出 MCP server 的版本兼容矩阵、失败时的错误形态（例如 key 无效返回什么）、以及 `PLUNK_ALLOW_UNCONFIRMED_SENDS` 之外的无提示方案；限流相关提示指向 API overview 的 1000 req/min，而自托管另有 token bucket 口径（见 `plunk-self-hosting`），两者关系文档未说明。
11. **高危/难撤销操作**：`plunk_send_campaign` 一旦确认无法召回（`plunk_cancel_campaign` 只拦未发出的部分）；`plunk_delete_contact` 直接删除档案；密钥轮换同时替换两个 key，所有在用集成需同步更新。
