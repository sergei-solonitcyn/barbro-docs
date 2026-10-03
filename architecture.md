# BarBro: Architecture

Updated: 2026-09-26
Status: expert council review passed 2026-09-26; revisions applied
Decisions: ADR-0001 … ADR-0006; threats and controls — `threat-model.md`.

## 1. Context (C4 Context)

```mermaid
flowchart TB
  user([User<br/>novice with a home bar,<br/>phone or laptop])
  barbro[["BarBro<br/>web app: bar, available cocktails,<br/>recommendation by context"]]
  google[Google<br/>OpenID Connect]
  deepl[DeepL API<br/>uk ⇄ en]
  llm[LLM API<br/>selection and explanation, EN]
  user -->|HTTPS| barbro
  barbro -->|sign-in| google
  barbro -->|free-text translation| deepl
  barbro -->|candidates + context → chosen ids + explanations| llm
```

BarBro has no inbound integrations: nobody calls its API except its own frontend. All three external systems are outbound, keyed, and none is required for FR-3a: the list of available cocktails works even if DeepL and the LLM are down.

## 2. Containers (C4 Container)

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

| Container  | Responsibility                                                                                                                                                                                                                                                                                                          | Technology                                       |
|------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| SPA        | Screens: sign-in, bar, available cocktails with filters, recommendation by context, recipe, account. Server state — TanStack Query; PWA for "add to home screen". Renders explanations as plain text only.                                                                                                              | Vite, React, TanStack Query, Tailwind, shadcn/ui |
| API        | The single public entry point. Modules: `auth` (OIDC, sessions), `catalog` (ingredients, tree, search), `bar` (the user's bar), `recipes` (recipes, matching, filters), `recommend` (FR-3b: limits, Translator, Sommelier, response validation), `i18n` (translations by locale). OpenAPI is generated from decorators. | NestJS + Fastify adapter, Drizzle, Zod           |
| SQLite     | A single database file in a Docker volume; WAL mode.                                                                                                                                                                                                                                                                    | better-sqlite3                                   |
| Litestream | Continuous WAL replication to R2; snapshot before a migration.                                                                                                                                                                                                                                                          | Litestream                                       |
| Caddy      | TLS (Cloudflare origin certificate), security headers (CSP, HSTS), SPA static files, proxying `/api` → API.                                                                                                                                                                                                             | Caddy                                            |

Adapters are the boundary for swapping providers without touching the modules:

- `Translator.translate(text, from, to) → text` — DeepL implementation; Azure/Google — by configuration.
- `Sommelier.recommend(candidates, context) → {picks: [{id, reason}]}` — implementation on the official SDK of the chosen model; model and provider — configuration; structured output; `max_tokens`; reasoning off.

## 3. Key flows

### 3.1 Sign-in via Google

```mermaid
sequenceDiagram
    participant B as Browser
    participant A as API
    participant G as Google
    B ->> A: GET /api/auth/google/start
    A ->> A: state, nonce, PKCE verifier → cookie
    A -->> B: 302 to Google authorize
    B ->> G: consent
    G -->> B: 302 /api/auth/google/callback?code&state
    B ->> A: callback
    A ->> A: verify state
    A ->> G: token endpoint: exchange code (PKCE verifier)
    G -->> A: id_token
    A ->> A: verify nonce, signature, email_verified
    A ->> A: upsert user + user_identity, create session
    A -->> B: Set-Cookie session (httpOnly, Secure, Lax), 302 /
```

### 3.2 "What to make now" (FR-3a, deterministic)

```mermaid
sequenceDiagram
  participant B as Browser
  participant A as API
  participant D as SQLite
  B->>A: GET /api/recipes/available?filter=light
  A->>D: user's bar (available) + default pantry
  A->>D: recipes with composition, satisfies_parent edges, substitutions
  A->>A: for each recipe: all non-optional positions covered? (ADR-0004) → makeable, approximate
  A->>A: filter by tags / ABV / method
  A->>D: translations for locale=uk
  A-->>B: list {id, name, tags, abv, approximate, why} — p95 < 500 ms
```

Matching is computed on every request: 663 recipes × ~5 positions — milliseconds in memory; no cache needed. The same code with a counter of uncovered positions gives S4 after release.

### 3.3 Recommendation by context (FR-3b)

```mermaid
sequenceDiagram
  participant B as Browser
  participant A as API
  participant T as DeepL
  participant L as LLM
  B ->> A: POST /api/recommend {context: "with a cheese board, for guests"}
  A ->> A: per-user limit 10/day and global 200/day, length ≤ 200
  A ->> A: candidates = FR-3a without a filter
  A->>T: translate(context, uk→en)
  A->>L: recommend(candidates[name, tags, abv, short description], context_en)
  L -->> A: {picks: [{id, reason_en}]} (structured)
  A ->> A: drop ids outside the candidates, cap at 3–5
  A->>T: translate(reasons, en→uk), cocktail names protected from translation
  A->>A: increment counters
  A-->>B: {picks: [{id, reason_uk}], remaining_today} — p95 < 10 s
```

Order matters: counters are incremented **after** successful paid calls, but the global check happens before them; on a DeepL or LLM error the user gets a clear response and the quota is not consumed.

## 4. Data model

```mermaid
erDiagram
  ingredient ||--o{ ingredient : parent
  ingredient ||--o{ recipe_ingredient : used_in
  recipe ||--|{ recipe_ingredient : has
  recipe }o--o{ tag: recipe_tag
  ingredient ||--o{ substitution: "from"
  ingredient ||--o{ substitution: "to"
  user ||--|{ user_identity: has
  user ||--o{ user_bar_item : owns
  ingredient ||--o{ user_bar_item : is
  user ||--o{ session : has
  user ||--o{ usage_counter : has

  ingredient {
    text id PK
    text name
    text parent_id FK
    text category
    real abv
    bool is_pantry
    bool satisfies_parent
  }
  substitution {
    text from_id FK
    text to_id FK
    text quality "exact | approximate"
  }
  recipe {
    text id PK
    text name
    text method
    text glass
    real abv
    text status "imported | reviewed"
    text source_url
    text description
  }
  recipe_ingredient {
    text recipe_id FK
    text ingredient_id FK
    real amount
    text unit
    bool optional
    text note
  }
  tag { text id PK }
  translation {
    text entity
    text entity_id
    text field
    text locale
    text text
  }
  user {
    text id PK
    text email
    datetime created_at
  }
  user_identity {
    text user_id FK
    text provider
    text subject
  }
  session {
    text id_hash PK
    text user_id FK
    datetime expires_at
    datetime absolute_expires_at
  }
  user_bar_item {
    text user_id FK
    text ingredient_id FK
    text status "available | out"
  }
  usage_counter {
    text scope "user:<id> | global"
    date day
    int count
  }
```

Reference tables (`ingredient`, `recipe`, `recipe_ingredient`, `tag`, `substitution`, `translation`) are populated by a seed script from Bar Assistant data (ADR-0001) with the edge and substitution audit (ADR-0004) and the nomenclature translation (ADR-0003); they are versioned together with the code. User tables are only `user`, `user_identity`, `session`, `user_bar_item`, `usage_counter`; account deletion is a single transaction over them. Pantry: no `user_bar_item` row for an `is_pantry` ingredient = available, `status = out` = disabled by the user. `usage_counter` counters reset at midnight UTC (02:00–03:00 Kyiv time) — the UI shows the reset time.

## 5. Deployment and operations

```mermaid
flowchart LR
  dev[SS: git push main] --> gha[GitHub Actions<br/>lint, test, audit, gitleaks, build]
  gha --> ghcr[(GHCR images api, caddy+spa)]
  vps[Hetzner CX23<br/>systemd timer: deploy the green HEAD of main<br/>by digest, health check, rollback] -->|pull| ghcr
  vps --> r2[(R2 backups)]
  vps -.-> sentry[Sentry]
  ur[UptimeRobot] -.-> vps
  ur -.-> tg[Telegram]
  vps -.->|CI run status| gha
  
```

- One compose file: `caddy`, `api`, `litestream`; a volume for SQLite; secrets in `.env` with mode 600.
- Deploy is pull-based (ADR-0007): every 5 minutes a systemd timer on the server takes the HEAD of `main` whose CI run
  succeeded, resolves the `sha-<commit>` tags of both images to digests, fetches `infra/` of the same commit, and runs `docker
  compose up -d` pinned by digest; a health check rolls back to the previous digests on failure; CI has no access to the
  server. Manual rollback or freeze — a pin file with the SHA.
- Monorepo `barbro`: `apps/api`, `apps/web`, `packages/shared`, `infra/`; documentation — `barbro-docs`.
- Hetzner firewall: 22 open to the internet with key-only authentication (accepted risk, see the threat model); 80/443
  from Cloudflare ranges.
- Drizzle migrations run at `api` start-up after a Litestream snapshot.
- Logs: pino → stdout → Docker `local` log driver (rotated by size, compressed); errors → Sentry.
- Restore: `litestream restore` into an empty volume — the procedure is in `runbook.md`, verified in phase 5.

Cost: server ≈ €6, domain ≈ $1.5, DeepL $0 until the 1 million characters are exhausted, LLM ≈ $0.5–3 — ≈ $9–11 per month in total.

## 6. Deliberately absent

Cache, queues, a separate worker, a second replica, a managed DB, an LLM gateway, streaming — each has a price in complexity that the NFRs do not justify. Each has a clear path to appear if phase 6 brings the corresponding requirement.
