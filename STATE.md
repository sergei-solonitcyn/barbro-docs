# STATE — BarBro

Updated: 2026-09-26

## Phase and milestone

Phase 2 Architecture — closed: the package (requirements, ADR-0001…0006, threat model, architecture) passed the expert council review; revisions applied; all documents rewritten in English for the public repo. Phase 3 Implementation — 0%; next milestone M0 "Skeleton": monorepo, CI, deploy of an empty application to the server with TLS.

## Done

- Measured the coverage of four recipe sources on the reference bar; chose Bar Assistant data (ADR-0001).
- Tested translators on domain texts: LibreTranslate unusable, DeepL usable (ADR-0003).
- Hosting survey: 12 providers, including Ukrainian ones (ADR-0006).
- Documents in `barbro-docs`, English: `README.md`, `requirements.md` (v2, FR-3 split), `adr/0001`–`0006` with `adr/README.md` index, `threat-model.md`, `architecture.md`, `repos.md`, `SECURITY.md`, `STATE.md`. English versions replace the Ukrainian originals (kept in git history).
- Hetzner account registered; CX23 available. DeepL API Developer key created (regenerate after the in-chat tests).

## Decisions

- ADR-0001 — Bar Assistant data (MIT) as the seed: all 663 recipes, status imported/reviewed, steps and descriptions rewritten by an LLM, no images.
- ADR-0002 — the LLM only for FR-3b: a sommelier over a deterministic list; FR-3a without the LLM.
- ADR-0003 — English data, i18n table; free text via DeepL (successor — Azure F0); no streaming; LLM behind an adapter, model by eval, fallback GPT-6 Luna.
- ADR-0004 — catalog tree for navigation, matching via `satisfies_parent` + a substitution table; default pantry.
- ADR-0005 — sign-in only via Google OIDC, server-side sessions in an httpOnly cookie, `user_identity` separate.
- ADR-0006 — TypeScript, Node 24 LTS, NestJS/Fastify, Vite + React SPA, SQLite + Drizzle, Hetzner CX23 + Docker Compose + Caddy, GitHub Actions + GHCR with pull deploy, Sentry + UptimeRobot, Litestream → R2, Cloudflare DNS; monorepo `barbro`.
- Licenses: `barbro` — AGPL-3.0, `barbro-docs` — CC BY 4.0. Accepted risk: AGPL reduces reuse of the code by people reading the repo as a sample; changeable while SS is the only author.
- Documentation norms: American English (en-US); `markdownlint-cli2` with MD013 off, other default rules on; `cspell` with `en-US` and a project word list.
- Program level (in methodology.md): repos public after phase 2 closes, repo language English, public repo standard (README, LICENSE, SECURITY.md, markdownlint, gitleaks, Conventional Commits, squash-merge).

## Stack and tools

Node 24 LTS, TypeScript, pnpm; NestJS on Fastify + nestjs-zod + OpenAPI; Vite, React, TanStack Query, Tailwind, shadcn/ui, PWA; SQLite (better-sqlite3), Drizzle, Litestream; Docker Compose, Caddy; GitHub Actions, GHCR; Sentry, UptimeRobot; Cloudflare DNS/proxy/R2; Biome, Vitest, Playwright, gitleaks, Renovate; Bruno for the API. Diagrams — Mermaid in Markdown. Adapters: `Translator` (DeepL → Azure), `Sommelier` (provider SDK).

## Open questions

- LLM model — by the eval set (~20 EN cases, with injection); candidates GPT-6 Luna, Gemini 3.5 Flash-Lite, Claude Haiku 4.5.
- Azure Translator F0 — test on the same phrases before phase 3 (needs a card).
- Domain.
- 18+ for an alcohol site (Legal, before phase 4). Privacy policy with the list of processors. "Source" link in the UI footer (AGPL section 13) — phase 4 checklist.
- Protecting cocktail names in DeepL (`tag_handling`) — verify at implementation.
- Google Gemini prices — verify on the official page before the eval.
- Public switch of both repos: CI for `barbro-docs` (markdownlint-cli2 + cspell) in progress; `LICENSE` (CC BY 4.0 legal code) in `barbro-docs`; then one review pass per the public repo checklist. Private vulnerability reporting on `barbro` — enabled.
- Organizational: Claude Code in a clone of the repo for reviews — optional.

## Learned in this product

- The expert council as a format: options with trade-offs, choice by the architect, risks in the ADR — practiced on six decisions.
- Measurements instead of assumptions: source coverage, MT quality, prices — every decision rested on a run or a verified source; twice "it seems" did not match the facts (DeepL Free is gone, Ukrainian VPS are not cheaper).
- Taxonomy ≠ interchangeability (catalog model); denial of wallet as the main threat of a free AI service; validating the LLM's structured output against the candidate list.
- Reviewing our own package found three blockers — contradictions between documents are visible only when they are read together.
- Writing documents for a public repo from the start is cheaper than translating them afterwards; the translation pass itself surfaced a stale "open questions" section in `requirements.md`.

## Next step

1. SS: finish CI for `barbro-docs` (`.markdownlint-cli2.jsonc`, `cspell.json`, GitHub Actions workflow), add `LICENSE`, commit the lint-clean package, Sync the Project knowledge; review pass; make both repos public.
2. New chat "Phase 3, M0: Skeleton". Before code — theory per the methodology: NestJS modules and DI, the event loop and synchronous better-sqlite3, the OIDC flow. Then: monorepo, CI with gitleaks/Biome/Vitest, images in GHCR, server with firewall and pull deploy, Caddy + TLS via Cloudflare, "hello" on the domain. M0 criterion — an empty application reachable over HTTPS, deployed from `main` hands-off.
3. In parallel, SS: domain, Azure F0 test, the LLM eval set.
