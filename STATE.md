# STATE — BarBro

Updated: 2026-09-27

## Phase and milestone

Phase 3 Implementation, milestone M0 "Skeleton" — about 10%. The hands-on part of the theory is done in a throwaway sandbox (NestJS 12 + Fastify + Vitest, outside-in TDD, validation); the `barbro` repo itself is not scaffolded yet. Phase 2 is closed; the status of the public-switch review pass and of making both repos public is not confirmed yet.

## Done

- Measured the coverage of four recipe sources on the reference bar; chose Bar Assistant data (ADR-0001).
- Tested translators on domain texts: LibreTranslate unusable, DeepL usable (ADR-0003).
- Hosting survey: 12 providers, including Ukrainian ones (ADR-0006).
- `barbro-docs` in English (en-US): `README.md`, `requirements.md` (v2, FR-3 split), `adr/0001`–`0006` with `adr/README.md` index, `threat-model.md`, `architecture.md`, `repos.md`, `SECURITY.md`, `STATE.md`, `LICENSE` (CC BY 4.0).
- CI for `barbro-docs`: `lint.yaml` with two jobs (markdownlint-cli2, cspell with en-US + Ukrainian dictionary), actions pinned by SHA, `permissions: contents: read`, Node from `.nvmrc`, tools via `package.json` + lock + `npm ci`. Dependabot for `github-actions` and `npm`. Ruleset on `main`: both checks required (source GitHub Actions), branches up to date, no bypass; changes only via PR with squash-merge. Verified with a test PR: red on a typo and on a Markdown error, merge blocked.
- `barbro`: `LICENSE` AGPL-3.0; private vulnerability reporting enabled.
- Hetzner account registered; CX23 available. DeepL API Developer key created (regenerate after the in-chat tests).
- M0 sandbox (outside the repos, to be discarded): `nest new` (ESM, Vitest), switched Express → Fastify, removed the scaffold samples; built `GET /health` and `POST /abv` test-first, outside-in: HTTP tests through `AppModule` + `inject`, service unit tests, a shared adapter factory and a `createTestApp` helper, Zod validation through `StandardSchemaValidationPipe` registered as `APP_PIPE`. All tests green.
- Cheat sheet (Claude Doc): [NestJS 12 + Fastify + Vitest, outside-in TDD](https://claude.ai/code/artifact/508f0c8f-e022-4509-bad7-1ded9016eb25) — setup, Nest concepts, TS/ESM pitfalls, test setup, TDD rules, validation, common errors, Python equivalents.

## Decisions

- ADR-0001 — Bar Assistant data (MIT) as the seed: all 663 recipes, status imported/reviewed, steps and descriptions rewritten by an LLM, no images.
- ADR-0002 — the LLM only for FR-3b: a sommelier over a deterministic list; FR-3a without the LLM.
- ADR-0003 — English data, i18n table; free text via DeepL (successor — Azure F0); no streaming; LLM behind an adapter, model by eval, fallback GPT-6 Luna.
- ADR-0004 — catalog tree for navigation, matching via `satisfies_parent` + a substitution table; default pantry.
- ADR-0005 — sign-in only via Google OIDC, server-side sessions in an httpOnly cookie, `user_identity` separate.
- ADR-0006 — TypeScript, Node 24 LTS, NestJS/Fastify, Vite + React SPA, SQLite + Drizzle, Hetzner CX23 + Docker Compose + Caddy, GitHub Actions + GHCR with pull deploy, Sentry + UptimeRobot, Litestream → R2, Cloudflare DNS; monorepo `barbro`.
- Licenses: `barbro` — AGPL-3.0, `barbro-docs` — CC BY 4.0. Accepted risk: AGPL reduces reuse of the code by people reading the repo as a sample; changeable while SS is the only author.
- Documentation norms: American English (en-US); markdownlint-cli2 with MD013 off; cspell en-US + `@cspell/dict-uk-ua`.
- CI supply chain: third-party actions pinned by full commit SHA with the exact version in a comment; `GITHUB_TOKEN` read-only by default; update bots — Dependabot in `barbro-docs` (only actions and two npm tools; no third-party app with write access), Renovate in `barbro` (pnpm workspaces, versions in Dockerfile URLs via regex managers, automerge) — set up in M0.
- Model and effort: routine config review — Medium (added to methodology.md).
- Program level (in methodology.md): repos public after phase 2 closes, repo language English, public repo standard.
- API code conventions (easily reversible, no ADR): NestJS 12 ESM project layout; Fastify router with `ignoreTrailingSlash: true`, set only in a shared adapter factory used by `main.ts` and tests; global pipes, guards, interceptors and filters registered as `APP_*` providers in modules, not via `app.useGlobal*()`; HTTP tests colocated as `*.spec.ts`, run in-process via `inject` (no supertest); one feature — one Nest module, `AppModule` only imports feature modules; TDD outside-in (HTTP test first, then service unit tests; thin controllers are not unit-tested); units in field names (`volumeMl`, `abvPercent`).

## Stack and tools

Node 24 LTS (26 under consideration, see open questions), TypeScript, pnpm 12 (Homebrew locally; version pinned via `packageManager`); NestJS 12 (ESM) on Fastify 5 + Zod through Nest's built-in Standard Schema validation + OpenAPI; Vite, React, TanStack Query, Tailwind, shadcn/ui, PWA; SQLite (better-sqlite3), Drizzle, Litestream; Docker Compose, Caddy; GitHub Actions, GHCR; Sentry, UptimeRobot; Cloudflare DNS/proxy/R2; Biome, Vitest 4 (handles Nest decorator metadata out of the box, no SWC plugin — verified in the sandbox), fast-check for property-based tests, Playwright, gitleaks, Renovate; Bruno for the API. Docs repo: markdownlint-cli2, cspell, Dependabot. Diagrams — Mermaid in Markdown. Adapters: `Translator` (DeepL → Azure), `Sommelier` (provider SDK) — DI tokens as abstract classes.

## Open questions

- Node 24 or 26: Node 26 enters LTS in October 2026 with EOL April 2029 (Node 24 — April 2028), which matches the ADR-0006 criterion "longest support at the start". Decide at scaffolding; if 26, add a note to ADR-0006.
- `nestjs-zod`: NestJS 12 validates Zod schemas natively (`@Body({ schema })` + `StandardSchemaValidationPipe`); `@nestjs/swagger` 12 can reflect them into OpenAPI via `standardSchemaConverter`. Confirm dropping `nestjs-zod` and check response serialization needs.
- Lint/format: Nest 12 generates oxlint + prettier; our choice is Biome. Configure Biome so `useImportType` autofix does not turn DI tokens into `import type`.
- TypeScript version: the Nest 12 generator uses TS 6; TS 7 reportedly has no programmatic compiler API, so `nest build` does not run on it. Verify at scaffolding.
- Local Node version management (Homebrew `node` follows the latest major): fnm, mise or Homebrew `node@N` — decide at scaffolding.
- Public-switch review pass and making both repos public — status to confirm.
- OIDC theory: proposed to move it to the start of the auth milestone (no auth in M0); confirm.
- LLM model — by the eval set (~20 EN cases, with injection); candidates GPT-6 Luna, Gemini 3.5 Flash-Lite, Claude Haiku 4.5.
- Azure Translator F0 — test on the same phrases before the i18n milestone (needs a card).
- Domain — blocks the Caddy + TLS step of M0.
- 18+ for an alcohol site (Legal, before phase 4). Privacy policy with the list of processors. "Source" link in the UI footer (AGPL section 13) — phase 4 checklist.
- Protecting cocktail names in DeepL (`tag_handling`) — verify at implementation.
- Google Gemini prices — verify on the official page before the eval.
- Dependabot: confirm both ecosystems run without errors and the first PR has a correct commit title (`ci(deps): …`, `chore(deps-dev): …`).
- Organizational: Claude Code in a clone of the repo for reviews — optional.

## Learned in this product

- The expert council as a format: options with trade-offs, choice by the architect, risks in the ADR — practiced on six decisions.
- Measurements instead of assumptions: source coverage, MT quality, prices, and in CI — reproducing a config locally before judging it; twice "it seems" did not match the facts.
- Taxonomy ≠ interchangeability (catalog model); denial of wallet as the main threat of a free AI service; validating the LLM's structured output against the candidate list.
- Reviewing our own package found three blockers — contradictions between documents are visible only when they are read together.
- Writing documents for a public repo from the start is cheaper than translating them afterwards.
- CI supply chain: tags are mutable, SHAs are not; pinning actions while installing tools unpinned with `npm install -g` moves the same risk one level down. `--fix` in CI turns a check into a no-op for fixable rules. A required check is only real after a deliberately failing PR proves it blocks.
- GitHub specifics: workflows only in `.github/workflows/`; required checks are named by the job `name:`, added in Rulesets manually, source pinned to GitHub Actions.
- NestJS modules, controllers, providers and DI tokens — understood and practiced; tokens as abstract classes for adapters — theory only.
- TypeScript under ESM: `.js` in relative imports, `import type` for types in decorated signatures but never for DI tokens, structural typing — practiced.
- Outside-in TDD (double loop, red for the right reason, fake it, triangulation, tests on behavior) with Vitest and Fastify `inject` — practiced; property-based testing — theory only.
- Test app parity: configuration shared between `main.ts` and tests (adapter factory, `APP_*` providers) — learned from a failing case.
- Validation: pipes, Zod schema as the single source of rules and type, `NaN` → `null` as a silent failure mode — practiced.
- Learning format: a batch of theory up front did not work; hands-on steps with each concept introduced when the code needs it did.
- Not yet: the event loop and synchronous better-sqlite3, OIDC flow.

## Next step

1. SS: confirm the public-switch status of both repos and Dependabot.
2. New chat "Phase 3, M0: Skeleton — repo" on Opus 5.5 or Fable 5.1, High. Optional warm-up in the sandbox: a property-based test with fast-check.
3. Decide Node 24/26, TypeScript version, local Node version manager; confirm Biome and dropping `nestjs-zod`.
4. SS scaffolds the pnpm monorepo `barbro` (`apps/api` from `nest new` ESM, cleaned per the cheat sheet; `apps/web` Vite + React) with `GET /api/health` test-first; Claude gives direction and reviews. Then CI (Biome, Vitest, gitleaks, SHA-pinned actions, read-only token) with Renovate, images in GHCR, server with firewall and pull deploy, Caddy + TLS via Cloudflare. M0 criterion — an empty application reachable over HTTPS, deployed from `main` hands-off.
5. In parallel, SS: domain (blocks the TLS step), Azure F0 test, the LLM eval set.
