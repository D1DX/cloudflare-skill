---
name: cloudflare
description: Unified Cloudflare skill — Workers (Hono), R2, D1, KV, DNS, SSL/TLS, caching, WAF, Zero Trust, Terraform IaC, wrangler, and deployment. Auto-triggers on any Cloudflare task.
disable-model-invocation: false
user-invocable: true
argument-hint: "task description"
---

# Cloudflare — Unified Skill

Workers, R2, D1, KV, DNS, SSL, caching, WAF, Zero Trust, and Terraform.

**As of April 2026.** Hono v4+, Wrangler v3+, Terraform provider v4.40+ (v5 migration pending).

---

## 1. Method Decision Tree

```
I need to manage DNS records, SSL, caching, WAF
  → Terraform if managed as IaC (recommended for Zero Trust)
  → Cloudflare API for one-off changes
  → Dashboard for visual inspection only

I need to build an API or web service
  → Cloudflare Worker with Hono framework
  → Deploy via wrangler

I need file storage
  → R2 bucket (S3-compatible, no egress fees)

I need a database
  → D1 (SQLite at the edge)

I need fast key-value lookups
  → KV namespace (eventually consistent — NOT for counters or transactions)

I need to protect a service behind login
  → Zero Trust Access Application (Terraform)

I need to let external webhooks bypass Zero Trust
  → Zero Trust bypass application with bypass policy (Terraform)

I need infrastructure as code
  → Terraform with Cloudflare provider v4.40+
```

---

## 2. Workers + Hono

### Project Setup

```bash
npm create cloudflare@latest my-worker -- --framework=hono --ts
```

### wrangler.jsonc (key settings)

```jsonc
{
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2025-08-06",
  "workers_dev": false,  // ALWAYS false for production — workers.dev bypasses Zero Trust
  "d1_databases": [{ "binding": "DB", "database_name": "my-db", "database_id": "xxx" }],
  "r2_buckets": [{ "binding": "BUCKET", "bucket_name": "my-bucket" }],
  "kv_namespaces": [{ "binding": "KV", "id": "xxx" }]
}
```

### Auth Middleware (Pattern)

```typescript
// Zero Trust: reads header injected by Cloudflare edge
const email = c.req.header('Cf-Access-Authenticated-User-Email')
  ?? (c.env.ENVIRONMENT === 'development' ? 'dev@local' : null);

// API endpoints: X-API-Key header with bypass policy in Zero Trust
const key = c.req.header('X-API-Key');
```

---

## 3. D1 Database (SQLite)

### Key Patterns

```typescript
// SELECT one — returns object or null
const row = await c.env.DB.prepare('SELECT * FROM items WHERE id = ?').bind(id).first();

// SELECT many — returns { results: [] }
const { results } = await c.env.DB.prepare('SELECT * FROM items WHERE status = ?').bind('active').all();

// Batch (atomic — all succeed or all fail)
await c.env.DB.batch([
  c.env.DB.prepare('INSERT INTO items (name) VALUES (?)').bind('A'),
  c.env.DB.prepare('UPDATE counters SET count = count + 1 WHERE name = ?').bind('items'),
]);
```

### Migrations

```bash
npx wrangler d1 migrations create my-db add_table
npx wrangler d1 migrations apply my-db --local   # test locally
npx wrangler d1 migrations apply my-db --remote  # production
```

### Limits

- Max DB size: 10 GB. Max row: 1 MB. Max SQL: 100 KB.
- **No `ALTER TABLE DROP COLUMN`** on older SQLite — use create-copy-rename pattern.
- **`--remote` required** — `wrangler d1 execute` defaults to local `.wrangler/state`.

---

## 4. R2 Storage

### Presigned URLs (SigV4 via aws4fetch)

```typescript
import { AwsClient } from 'aws4fetch';

const client = new AwsClient({
  accessKeyId: env.R2_ACCESS_KEY_ID,      // separate from API token
  secretAccessKey: env.R2_SECRET_ACCESS_KEY,
});

const url = new URL(`https://${env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com/${env.R2_BUCKET_NAME}/${key}`);
url.searchParams.set('X-Amz-Expires', '604800');
const signed = await client.sign(new Request(url, { method: 'GET' }), { aws: { signQuery: true } });
```

### Multipart Upload

Min 5 MiB per part (except last). Start → uploadPart(n, chunk) → complete([parts]).

---

## 5. KV Namespace

```typescript
await c.env.KV.get('key');              // string or null
await c.env.KV.get('key', 'json');      // parsed JSON
await c.env.KV.put('key', value, { expirationTtl: 3600, metadata: {} });
await c.env.KV.delete('key');
await c.env.KV.list({ prefix: 'webhook/', limit: 100 });
```

**KV is eventually consistent** — reads after writes may be stale for up to 60 seconds. Don't use for counters or transaction state. Use D1 for transactional data.

---

## 6. DNS

### Proxied vs DNS-Only

| Setting | Proxied (orange cloud) | DNS-Only (grey cloud) |
|---------|----------------------|---------------------|
| Traffic | Through Cloudflare CDN | Direct to origin |
| DDoS | Protected | Exposed |
| IP | Cloudflare IPs | Origin IP exposed |
| Use when | Web traffic, APIs | Mail servers, non-HTTP |

**Rule:** Always proxy HTTP/HTTPS. DNS-only for MX, non-HTTP services.

### SSL/TLS Modes

| Mode | Use |
|------|-----|
| Full (Strict) | **Always use this** — origin has valid cert or Cloudflare origin cert |
| Full | Self-signed OK (avoid if possible) |
| Flexible | HTTP to origin — **insecure, never use** |

### Page Rules → Modern Replacements

Page Rules deprecated. Use: **Redirect Rules** (forwarding), **Cache Rules** (caching), **Transform Rules** (URL rewriting).

---

## 7. Zero Trust (Terraform)

### Access Application

```hcl
resource "cloudflare_zero_trust_access_application" "dashboard" {
  account_id                = var.account_id
  name                      = "Internal Dashboard"
  domain                    = "dashboard.example.com"
  type                      = "self_hosted"
  session_duration          = "24h"
  allowed_idps              = [cloudflare_zero_trust_access_identity_provider.google.id]
  auto_redirect_to_identity = true
  policies = [{ id = cloudflare_zero_trust_access_policy.allow_team.id, precedence = 1 }]
}
```

### Webhook Bypass Pattern

```hcl
resource "cloudflare_zero_trust_access_application" "webhook_bypass" {
  account_id       = var.account_id
  name             = "Bypass — Webhooks"
  domain           = "n8n.example.com/webhook/*"
  type             = "self_hosted"
  session_duration = "0s"
  policies = [{ id = cloudflare_zero_trust_access_policy.bypass_public.id, precedence = 1 }]
}
```

### Policy Decisions

`allow`, `deny`, `bypass`, `non_identity`. Order: Service Auth > Bypass > Allow > Block (first match wins).

Include/exclude: `email`, `email_domain`, `everyone`, `ip`, `service_token`, `group`, `geo`.

### for_each Pattern (Multiple Bypasses)

```hcl
variable "bypass_services" {
  type = map(object({ domain = string, name = string }))
}

resource "cloudflare_zero_trust_access_application" "bypass" {
  for_each         = var.bypass_services
  account_id       = var.account_id
  name             = each.value.name
  domain           = each.value.domain
  type             = "self_hosted"
  session_duration = "0s"
  policies = [{ id = cloudflare_zero_trust_access_policy.bypass_public.id, precedence = 1 }]
}
```

---

## 8. Terraform Provider

### Setup

```hcl
cloudflare = { source = "cloudflare/cloudflare", version = ">= 4.40.0, < 5.0.0" }
```

Auth: `CLOUDFLARE_API_TOKEN` env var (API Token, not API Key).

### Import Existing Resources

```bash
# cf-terraforming (official)
cf-terraforming generate --token "$TOKEN" --account "$ACCT" \
  --resource-type cloudflare_zero_trust_access_application > apps.tf

# Terraform 1.5+ import blocks
import { to = cloudflare_resource.name, id = "account_id/resource_id" }
```

### v5 Migration Status (April 2026)

v5.17.0 GA but has Zero Trust bugs (idempotency #5565, reusable policy #5499). **Stay on v4.40+ until migration tool ships.** Using `zero_trust_` prefixed resource names on v4 is forward-compatible with v5.

Key v4→v5 renames: `cloudflare_access_*` → `cloudflare_zero_trust_access_*`, `cloudflare_record` → `cloudflare_dns_record`.

---

## 9. Wrangler Commands

```bash
npx wrangler dev                   # local dev
npx wrangler dev --remote          # remote dev (real bindings)
npx wrangler deploy                # production
npx wrangler secret put API_KEY    # set secret
npx wrangler d1 execute db --command "SELECT 1" --remote
npx wrangler tail                  # real-time logs
```

---

## 10. Gotchas

### Workers
1. **`workers_dev = false`** — always for production. workers.dev URL bypasses Zero Trust.
2. **Zero Trust headers missing locally** — `Cf-Access-Authenticated-User-Email` not set in dev. Fallback needed.
3. **R2 presigned URLs need separate credentials** — R2_ACCESS_KEY_ID/SECRET, not regular API token.
4. **Hono `c.env` is per-request** — don't cache env outside handlers.
5. **Subrequest limit: 1000** — each D1/KV/R2 operation counts.
6. **Max Worker size: 10 MB** after bundling.

### D1
7. **D1 `batch()` is atomic** — all succeed or all fail.
8. **`first()` returns object or null. `all()` returns `{ results: [] }`.**
9. **Always use prepared statements** — `bind()` prevents injection.

### R2
10. **R2 CORS** — must configure on bucket itself, not just Worker response headers.

### DNS & SSL
11. **MX records cannot be proxied** — must be grey cloud. Email breaks if proxied.
12. **Origin certs are Cloudflare-only** — not trusted by browsers directly.
13. **TTL = 1 means "auto"** — Cloudflare sets optimal TTL.
14. **Wildcard DNS doesn't auto-proxy** — each subdomain needs individual records.

### Terraform
15. **Never manage same resource in Terraform AND dashboard** — state drift.
16. **State file contains secrets** — never commit. `.gitignore`: `*.tfstate`, `.terraform/`.
17. **R2 CORS/lifecycle needs AWS provider** — CF provider only creates/deletes buckets.
18. **D1 schema via Wrangler, not Terraform.**
19. **`for_each` ordering is random** — use explicit `precedence` for policies.
20. **v5 idempotency bug** — `exclude: []` vs `null` causes phantom updates.
21. **Use `-parallelism=5`** on apply if hitting API rate limits (429s).

