# ADR-0006: Stack and hosting

Status: accepted, amended (see Amendments)
Date: 2026-09-26

## Context

A program-level input constraint: the backend is **Node.js**, because BarBro is the only portfolio product where Node is chosen deliberately as a learning target; other products will mostly go to Python. From this follows the criterion: cover what the Node market values most, and choose a frontend that carries over to projects with a Python backend.

Product constraints: budget ≤ $20/month, ≈ $8–10 of it for hosting and domain; one developer; NFRs — 20 DAU, 5 rps peak, p95 < 500 ms, 99% availability, one server without failover, RPO 24 h. The data model (ADR-0004) needs recursive queries over the catalog tree. Repositories become public after phase 2 closes — this affects CI (free minutes) and the quality standard.

## Options

**Backend framework.** Fastify teaches Node itself best (plugins, hooks, schema validation), but per 2026 overviews the market for new enterprise services takes NestJS by default, Express remains the baseline, Fastify is a throughput niche. Hono — Web standards, closer to the edge than to Node. Next.js is not a backend framework but a React meta-framework with a BFF layer; it covers the "React full-stack" position, not "Node backend". Bundles considered: (A) Next.js full-stack; (B) NestJS API + React SPA; (C) NestJS + Next.js — maximum coverage, excessive overhead for a first product.

**Frontend model.** SPA + API — an explicit contract, the frontend carries over to any backend; an SSR framework — faster, but hides the backend; server-rendered HTML + HTMX — minimal JS, teaches less of the modern stack.

**Database.** SQLite on the same server: zero infrastructure, recursive CTEs available, backup — Litestream continuously; does not teach Postgres. Postgres managed (Neon, Supabase — free tiers) or in Docker: the industry standard, an extra link for 100 users; Postgres will inevitably appear in later portfolio products.

**Hosting** (prices excluding VAT, September 2026; official pages unless noted otherwise):

| Provider, plan                                       | vCPU / RAM / disk     | Price/month                                   | DC                                                      |
|------------------------------------------------------|-----------------------|-----------------------------------------------|---------------------------------------------------------|
| Hetzner CX23                                         | 2 / 4 GB / 40 GB      | €5.49 + ~€0.5 IPv4                            | Nuremberg, Falkenstein, Helsinki                        |
| OVHcloud VPS-1                                       | 2 / 4 GB / 40 GB NVMe | "from" $4.54 (with a contract)                | Warsaw, Frankfurt                                       |
| Netcup VPS 500                                       | 2 / 4 GB / 64 GB      | €8.26 (12 months)                             | Nuremberg                                               |
| Scaleway DEV1-S                                      | 2 / 2 GB / 20 GB      | ≈ €6.5                                        | Paris, Amsterdam, Warsaw                                |
| DigitalOcean / Linode / Vultr / UpCloud              | 1 / 1 GB              | $5–7; 4 GB — ≈ $24 (partly secondary sources) | Frankfurt, Warsaw                                       |
| ukraine.com.ua VPS 2G / 4G                           | 1 / 2 GB; 1 / 4 GB    | €6.15 / €12.66                                | Kyiv                                                    |
| HOSTiQ kVPS50, HostPro NVMe                          | 4 GB                  | from $13 / from €11                           | Rotterdam / Kyiv                                        |
| PaaS (Railway, Fly.io, Render)                       | —                     | from $5, usage-dependent                      | —                                                       |
| Serverless (Vercel Hobby + Neon, Cloudflare Workers) | —                     | $0                                            | Vercel Hobby — non-commercial only; edge runtime ≠ Node |

NestJS + SQLite + Caddy in Docker live comfortably in 2 GB provided builds run in CI. At 4 GB only Hetzner and OVH fit the budget; Ukrainian providers are twice as expensive for the same class and have no API/Terraform.

## Decision

| Layer             | Choice                                                                                                                                                                   | Why                                                                                                                                                                                                                                   |
|-------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Language, runtime | TypeScript, Node.js 24 LTS (active LTS until October 2026, maintenance until April 2028), pnpm                                                                           | TS is the ecosystem norm; the LTS with the longest support at the start                                                                                                                                                               |
| Backend           | **NestJS** on the Fastify adapter; OpenAPI from decorators; validation — Zod via `nestjs-zod` (class-validator is not used, to keep one schema for frontend and backend) | The market default for Node backends; the Fastify adapter gives speed and schema validation                                                                                                                                           |
| Frontend          | **Vite + React**, TanStack Query, Tailwind, shadcn/ui, PWA; Playwright for E2E on a mobile viewport                                                                      | An SPA over an explicit API — carries over to the Python backends of later products                                                                                                                                                   |
| DB                | **SQLite** (better-sqlite3) + **Drizzle**                                                                                                                                | Product size; zero infrastructure; Drizzle keeps SQL transparent and eases the future move to Postgres. `better-sqlite3` is synchronous — every query blocks the event loop; at our scale that is milliseconds, accepted deliberately |
| Hosting           | **Hetzner CX23**, Docker Compose, Caddy (automatic TLS). Fallback: CAX11 (ARM) or OVH VPS-1                                                                              | The cheapest 4 GB in the EU with an API, a Terraform provider and hourly billing; CX23 availability confirmed at sign-up                                                                                                              |
| CI/CD             | GitHub Actions → images in GHCR; **pull deploy**: the server itself polls GHCR (cron script or Watchtower) and restarts compose on a new digest                          | Free for a public repo; builds not on the server; the server accepts no inbound connections from CI — SSH only for SS                                                                                                                 |
| Observability     | Sentry free, pino → stdout → Docker logs, UptimeRobot (alerts to Telegram)                                                                                               | $0; enough for 99% and one server                                                                                                                                                                                                     |
| Backups           | Litestream → Cloudflare R2 (10 GB free)                                                                                                                                  | Continuous SQLite replication; RPO effectively minutes instead of 24 h                                                                                                                                                                |
| Domain, DNS       | Cloudflare DNS + proxy (WAF, rate limiting free)                                                                                                                         | First layer of defense for the threat model                                                                                                                                                                                           |
| Code quality      | Biome, Vitest, gitleaks in CI, Conventional Commits, squash-merge                                                                                                        | Public repo standard                                                                                                                                                                                                                  |
| External SDKs     | `@nestjs/*`, `openai` / `@anthropic-ai/sdk` / `@google/genai`, `deepl-node` — behind the adapters from ADR-0003                                                          | No gateway layer                                                                                                                                                                                                                      |

Plain Fastify was dropped not for its shortcomings but on the market criterion; Next.js — because it is a frontend position, not a backend one, and an SPA carries over better.

## Consequences

- We get: a set where every layer is either a market default (NestJS, React, Tailwind) or "boring tech" for the size (SQLite, one VPS); the full SRE dimension in phase 5 (deploy, backups, restore, incident — all on our own machine); cost ≈ €6 server + ≈ $1.5 domain.
- We pay: NestJS — DI, decorators, modules — overhead for three screens, accepted deliberately as a learning subject; two builds (API and SPA) and CORS/cookie nuances; SQLite → Postgres will be a migration one day.
- Accepted risks: Hetzner raised prices twice in 2026 — the budget has margin; one server — 99%, no more, as in the NFRs; Litestream is a third-party tool on the critical backup path, restore is verified in phase 5 without fail.
- Repositories: monorepo `barbro` with pnpm workspaces — `apps/api`, `apps/web`, `packages/shared` (Zod schemas and types shared by frontend and backend), `infra/` (compose, Caddyfile, server scripts); `barbro-docs` separately, per the methodology.
- Open (in STATE, not in the ADR): the code license of the public repo — MIT or AGPL; the concrete domain.

## Amendments

### 2026-09-27: runtime, validation glue, open items

- **Runtime: Node.js 26 LTS instead of 24.** Node.js 26 is promoted to LTS in October 2026 and reaches end of life in April 2029, a year later than Node.js 24 (April 2028). By this ADR's own criterion, the LTS with the longest support at the start, it wins. At scaffolding time it is still Current for a few weeks; production (phase 4) starts after the LTS promotion. The main compatibility risk for a native module is gone: better-sqlite3 13 is built on Node-API, so its prebuilt binaries no longer depend on the Node.js ABI.
- **Validation: NestJS built-in Standard Schema support instead of `nestjs-zod`.** NestJS 12 validates Zod schemas natively (`@Body({ schema })` with `StandardSchemaValidationPipe`), and `@nestjs/swagger` 12 reflects them into OpenAPI via `standardSchemaConverter`. The decision itself (one Zod schema shared by frontend and backend, no class-validator) is unchanged; only the glue library is dropped. Response serialization is revisited at the first endpoint with a real contract.
- **Open items.** The code license is resolved: AGPL-3.0 (see `STATE.md`). The domain is still open.

### 2026-10-04: ingress through Cloudflare Tunnel; domain

Replaces "Caddy (automatic TLS)" in the Hosting row and the inbound 80/443 assumption of the threat model. The rest of the decision is unchanged.

**Context.** The architecture put Cloudflare in front of the server, so the origin must accept web traffic only from Cloudflare and hold a certificate Cloudflare trusts. At implementation three ways to do that were compared:

| Option                                                                | Open on the server                | Secret on the server                                                                                                                         | Cloudflare IP ranges                                      | Edge image                       |
|-----------------------------------------------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|----------------------------------|
| A. Cloudflare origin certificate + firewall by Cloudflare ranges      | 22; 80/443 from Cloudflare ranges | origin certificate key; for automatic range refresh, a Hetzner API token, which is project-wide and can also delete the server and snapshots | in two places: the firewall and Caddy's `trusted_proxies` | official                         |
| B. Let's Encrypt via DNS-01 + firewall by Cloudflare ranges           | 22; 80/443 from Cloudflare ranges | Cloudflare API token with DNS edit rights — whoever holds it can redirect the domain                                                         | same as A                                                 | custom build with a DNS plugin   |
| C. Cloudflare Tunnel: `cloudflared` opens an outbound connection only | 22                                | tunnel token — whoever holds it can attach a connector and receive a share of the traffic                                                    | not needed                                                | official; TLS ends at Cloudflare |

**Decision.** Option C, a remotely managed tunnel: the tunnel and its single rule (all traffic for `barbro.dev` → `http://edge:8080`) live in the Cloudflare dashboard; routing, security headers and compression stay in the edge Caddyfile in `infra/`. Domain: `barbro.dev`, registered with Cloudflare Registrar (at-cost pricing, the zone on Cloudflare nameservers from the start, DNSSEC enabled).

**Consequences.**

- We get: no inbound HTTP ports at all — the Hetzner firewall keeps a single rule for SSH, and direct access to the origin bypassing Cloudflare is closed by construction; no Cloudflare range list to keep in sync and no Hetzner API token on the server; no certificate on the origin; the real client IP needs to be trusted from one known peer (the `tunnel` container) instead of a list of ranges.
- We pay: a fourth container (`cloudflared`, pinned by digest, updated by Renovate); the first secret on the server (the tunnel token, a Compose file-based secret, see `infra/README.md` in `barbro`); ingress depends on Cloudflare entirely — leaving Cloudflare means implementing option A or B first.
- Accepted risks: a leaked tunnel token lets an attacker run a connector and receive part of the users' traffic, sessions included — mitigated by keeping the token only in a root-owned file readable by the container's group, and rotating it on any suspicion; the deploy agent's health check goes through the edge on localhost and does not see the tunnel, so a broken tunnel stays unnoticed until external monitoring exists (UptimeRobot, phase 5).
- The domain open item is resolved.
