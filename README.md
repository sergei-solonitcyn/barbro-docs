# BarBro — documentation

BarBro is a web app for the home bar: you keep a list of the bottles and ingredients you have, see which cocktails you can make right now from a database of real, verified recipes, and ask for a recommendation for a mood, an occasion or a dish. That last step is the only place an LLM is involved, and it can only choose among cocktails you can already make — it never invents a recipe. Built for someone who likes making cocktails at home; the first product of a Solution Architect portfolio program, training the Data / AI dimension.

```mermaid
flowchart TB
  user([User])
  subgraph cf[Cloudflare]
    proxy[DNS + proxy + WAF + rate limit]
  end
  subgraph vps[Hetzner CX23 · Docker Compose]
    caddy[Caddy<br/>TLS, SPA static files, reverse proxy]
    api[API<br/>NestJS on Fastify, TypeScript<br/>auth, bar, matching, limits,<br/>Translator and Sommelier adapters]
    db[(SQLite<br/>catalog, recipes, translations,<br/>users, bar, counters)]
    ls[Litestream<br/>WAL replication]
  end
  spa[SPA<br/>Vite + React, PWA<br/>runs in the browser]
  r2[(Cloudflare R2<br/>backups)]
  google[Google OIDC]
  deepl[DeepL]
  llm[LLM]
  sentry[Sentry]
  user --> proxy --> caddy
  caddy -->|/ static files| spa
  caddy -->|/api| api
  api --> db
  ls --> db
  ls --> r2
  api --> google
  api --> deepl
  api --> llm
  api -.errors.-> sentry
```

## Documents

| Document                             | What it is                                                                       |
|--------------------------------------|----------------------------------------------------------------------------------|
| [`requirements.md`](requirements.md) | Problem, users, scenarios, MVP scope, functional and non-functional requirements |
| [`architecture.md`](architecture.md) | C4 Context and Container diagrams, key flows, data model, deployment             |
| [`adr/`](adr/README.md)              | Architecture decision records, one file per decision                             |
| [`threat-model.md`](threat-model.md) | STRIDE over the data flows: assets, boundaries, threats and controls             |
| [`repos.md`](repos.md)               | The product's repositories and what lives in each                                |
| [`STATE.md`](STATE.md)               | Current phase, milestone, decisions and next step                                |

Code lives in the [`barbro`](repos.md) monorepo.

## Status

Phase 2 (architecture) is closed; phase 3 (implementation) starts with milestone M0 "Skeleton". See `STATE.md`.
