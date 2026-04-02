# Cloudflare Skill

[![Author](https://img.shields.io/badge/Author-Daniel_Rudaev-000000?style=flat)](https://github.com/daniel-rudaev)
[![Studio](https://img.shields.io/badge/Studio-D1DX-000000?style=flat)](https://d1dx.com)
[![Cloudflare](https://img.shields.io/badge/Cloudflare-Skill-F38020?style=flat&logo=cloudflare&logoColor=white)](https://cloudflare.com)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat)](./LICENSE)

Unified Cloudflare skill for AI agents. Covers Workers (Hono), R2, D1, KV, DNS, SSL/TLS, caching, WAF, Zero Trust, Terraform IaC, and wrangler deployment. Built from production infrastructure work.

## What's Included

| Topic | What it covers |
|-------|---------------|
| Method Decision Tree | When to use Workers vs R2 vs D1 vs KV vs Zero Trust vs Terraform |
| Workers + Hono | Project setup, wrangler.jsonc, auth middleware, Zero Trust header pattern |
| D1 Database | SELECT one/many, batch (atomic), migrations, wrangler commands, limits |
| R2 Storage | Presigned URLs (SigV4 via aws4fetch), multipart upload |
| KV Namespace | Get, put (with TTL), delete, list — eventual consistency caveats |
| DNS | Proxied vs DNS-only, SSL/TLS modes, Page Rules replacements |
| Zero Trust (Terraform) | Access applications, webhook bypass pattern, policy decisions, for_each |
| Terraform Provider | Setup, import existing resources, v4→v5 migration status |
| Wrangler Commands | dev, deploy, secrets, D1 execute, tail logs |
| Gotchas | 21 hard-won lessons across Workers, D1, R2, DNS, and Terraform |

## Install

### Claude Code

```bash
git clone https://github.com/D1DX/cloudflare-skill.git
cp -r cloudflare-skill ~/.claude/skills/cloudflare
```

Or as a git submodule:

```bash
git submodule add https://github.com/D1DX/cloudflare-skill.git path/to/skills/cloudflare
```

### Other AI Agents

Copy `SKILL.md` into your agent's prompt or knowledge directory. The skill is structured markdown — works with any LLM agent that reads reference files.

## Structure

```
cloudflare-skill/
└── SKILL.md    — Main skill (Workers, R2, D1, KV, DNS, Zero Trust, Terraform, gotchas)
```

## Sources

- **Workers, D1, R2, KV:** Verified against [Cloudflare Workers documentation](https://developers.cloudflare.com/workers/) and production deployments (April 2026).
- **Zero Trust & Terraform:** Verified against [Cloudflare Terraform provider docs](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs) and live Terraform configurations (v4.40+).
- **v5 migration status:** Verified from [Cloudflare provider changelog](https://github.com/cloudflare/terraform-provider-cloudflare/releases) (April 2026).

## Credits

Built by [Daniel Rudaev](https://github.com/daniel-rudaev) at [D1DX](https://d1dx.com).

## License

MIT License — Copyright (c) 2026 Daniel Rudaev @ D1DX
