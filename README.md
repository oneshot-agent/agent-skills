# OneShot Agent Skills

[![skills.sh](https://skills.sh/b/oneshot-agent/agent-skills)](https://skills.sh/oneshot-agent/agent-skills)

Agent Skills that teach any coding agent (Claude Code, Cursor, Codex, and
[70+ others](https://github.com/vercel-labs/skills#supported-agents)) how to use
[**OneShot**](https://oneshotagent.com) — infrastructure for autonomous AI agents to execute
real-world commercial actions (email, SMS, voice, physical mail, research, enrichment,
commerce, browser automation, website builds, and autonomous compute goals), priced per call and settled either
in **USDC on Base** via **x402** or by card through **Stripe's Agentic Commerce Protocol**.

## Install

```bash
# Install all OneShot skills into your agent
npx skills add oneshot-agent/agent-skills

# Or just what you need
npx skills add oneshot-agent/agent-skills --skill oneshot --skill oneshot-email

# List what's available
npx skills add oneshot-agent/agent-skills --list
```

Start with the **`oneshot`** skill — it covers install, wallet/auth (Coinbase CDP or raw key),
funding, **spend budgets**, and the shared options every paid tool accepts. Then add the
capability skills you need.

## Skills

| Skill | What it does |
|-------|--------------|
| [`oneshot`](skills/oneshot/SKILL.md) | **Start here.** Setup, auth, funding, x402 model, shared options, MCP server config (local, and the hosted endpoint for Grok Bot / cloud agents) |
| [`oneshot-email`](skills/oneshot-email/SKILL.md) | Send/receive email, attachments, reply threading, sending-domain pool & warmup |
| [`oneshot-messaging`](skills/oneshot-messaging/SKILL.md) | SMS send/inbox and autonomous AI voice calls |
| [`oneshot-research`](skills/oneshot-research/SKILL.md) | Deep cited research, web search, read-a-URL-as-markdown |
| [`oneshot-enrichment`](skills/oneshot-enrichment/SKILL.md) | People & company search, profile/company enrichment, find/verify email, deep person intelligence |
| [`oneshot-local`](skills/oneshot-local/SKILL.md) | Local business discovery by category × location; name + address → domain, phone, status |
| [`oneshot-gov`](skills/oneshot-gov/SKILL.md) | Federal solicitations (SAM.gov) by NAICS, with the contracting officer's contact |
| [`oneshot-physical-mail`](skills/oneshot-physical-mail/SKILL.md) | Printed letters and postcards: artwork upload, address validation, preview, approval, tracking |
| [`oneshot-commerce`](skills/oneshot-commerce/SKILL.md) | Product search and autonomous purchase |
| [`oneshot-browser`](skills/oneshot-browser/SKILL.md) | Autonomous browser tasks + persistent logged-in profiles |
| [`oneshot-build`](skills/oneshot-build/SKILL.md) | Generate & deploy websites, then update them |
| [`oneshot-compute`](skills/oneshot-compute/SKILL.md) | Multi-step autonomous goals with a budget + spend analytics |
| [`soul-markets`](skills/soul-markets/SKILL.md) | Monetize an agent: list a soul, sell/buy services, earn USDC |

## How it works

Each skill is a [`SKILL.md`](https://github.com/vercel-labs/skills#what-are-agent-skills) file —
YAML frontmatter (`name` + `description`) plus instructions your agent reads on demand. Skills
document the [`@oneshot-agent/sdk`](https://www.npmjs.com/package/@oneshot-agent/sdk) (TypeScript)
and [`@oneshot-agent/mcp-server`](https://www.npmjs.com/package/@oneshot-agent/mcp-server).

## Links

- OneShot: https://oneshotagent.com · Docs: https://docs.oneshotagent.com
- Cursor plugin (these tools plus spend-safety rules): https://github.com/tormine/oneshot-cursor-plugin
- Soul.Markets: https://soul.mds.markets
- Skills ecosystem: https://skills.sh

## License

MIT — see [LICENSE](LICENSE).
