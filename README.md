# Plunk Skills

A pack of 8 AI-agent skills distilled from the [Plunk](https://plunk.com) developer
documentation ([docs.useplunk.com](https://docs.useplunk.com), MIT-licensed source
[useplunk/plunk](https://github.com/useplunk/plunk)). Plunk is an open-source email
service (sending, receiving, contacts, segments, templates, campaigns, workflows,
self-hosting).

These skills turn the entire Plunk docs site into on-demand, versionable agent
context — so your coding agent can answer Plunk API, deliverability, workflow, and
self-hosting questions accurately instead of guessing from stale training data.

> **Language:** the skill *content* is written in Chinese (中文). Skill names,
> triggers, and code/API identifiers are in English. If you need an English version,
> fork and translate — the structure is plain Markdown.

## Install

With the [Vercel Skills CLI](https://skills.sh) (works for Claude Code, Cursor,
Codex, GitHub Copilot, WorkBuddy, and 18+ agents):

```bash
# install all 8 skills, globally, no prompts
npx skills add <your-github-username>/plunk-skills --yes --global

# or install just one
npx skills add <your-github-username>/plunk-skills --skill plunk-api-essentials --yes --global
```

The first install also registers the pack on [skills.sh](https://skills.sh) for
discovery. To install into the current project instead of globally, drop `--global`.

## What's inside

| Skill | Covers |
| --- | --- |
| `plunk` | **Router / entry point.** Routes any Plunk question to the right skill; also holds the shared glossary, decision cheat-sheet, and getting-started guide. |
| `plunk-api-essentials` | API keys (`pk_`/`sk_`), error envelope, idempotency keys, status-code ⇄ error-code map, pagination, rate limits. |
| `plunk-domain-deliverability` | Sending-domain DNS (SPF/DKIM/DMARC/bounce MX), SPF merge rule, "verified but still in spam" troubleshooting. |
| `plunk-contacts-and-fields` | `data.*` modeling, type inference & "type pollution", CSV import, reserved keys, deletion. |
| `plunk-templates-liquid` | MARKETING / TRANSACTIONAL / HEADLESS templates, Liquid rendering, `??` fallback, locale fallback. |
| `plunk-workflows` | Triggers, 8 step types, `branch`/`splice`, active-execution edit lock, "why isn't it firing" diagnosis. |
| `plunk-inbound-email` | Inbound MX, verdict (spam/virus/spf/dkim/dmarc), routing & auto-reply, loop protection. |
| `plunk-self-hosting` | Docker Compose, AWS SES (IAM/SNS), sandbox exit, retention/cleanup worker, env vars. |

The remaining 11 topics (segments & filters, subscription & consent, campaigns &
analytics, webhooks, transactional send, tracking events, sending health, recipes,
data retention, billing & limits, MCP & agents) live as capability cards **inside**
the `plunk` router and load on demand.

## Known doc gaps (kept as boundaries, not invented fixes)

- The Workflow **API** only ships a route table, not a full request schema — so
  building graphs purely via API is not promised; use the dashboard/MCP or probe
  `/workflows/fields`.
- Rate-limit is documented with **two** different figures (1000 req/min vs.
  self-hosted token-bucket default-off). Both are preserved.
- A success-response example in the docs contains `success: false` — flagged as a
  doc artifact; the contract follows the field description.

## Repository layout

```
plunk-skills/
├── skills/
│   ├── plunk/                  # router + references/ (glossary, capability cards, cheat-sheet)
│   ├── plunk-api-essentials/
│   ├── plunk-domain-deliverability/
│   ├── plunk-contacts-and-fields/
│   ├── plunk-templates-liquid/
│   ├── plunk-workflows/
│   ├── plunk-inbound-email/
│   └── plunk-self-hosting/
├── LICENSE
└── README.md
```

## License

MIT — see [LICENSE](./LICENSE). Source docs are MIT (useplunk/plunk).
