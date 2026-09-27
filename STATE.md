# STATE — BarBro

Updated: 2026-09-28

## Phase and milestone

Phase 3 Implementation, milestone M0 "Skeleton" — about 30%. The `barbro` monorepo is scaffolded: root workspace tooling
and `apps/api` (NestJS 12 ESM on Fastify) with `GET /api/health` built test-first and environment config validated at
startup. Remaining in M0: `apps/web`, the repo README, CI with Renovate, images in GHCR, the server with firewall and
pull deploy, Caddy + TLS. Both repos are public.

## Done

- Measured the coverage of four recipe sources on the reference bar; chose Bar Assistant data (ADR-0001).
- Tested translators on domain texts: LibreTranslate unusable, DeepL usable (ADR-0003).
- Hosting survey: 12 providers, including Ukrainian ones (ADR-0006).
- `barbro-docs` in English (en-US): `README.md`, `requirements.md` (v2, FR-3 split), `adr/0001`–`0006` with
  `adr/README.md` index, `threat-model.md`, `architecture.md`, `repos.md`, `SECURITY.md`, `STATE.md`, `LICENSE` (CC BY
  4.0).
- CI for `barbro-docs`: `lint.yaml` with two jobs (markdownlint-cli2, cspell with en-US + Ukrainian dictionary), actions
  pinned by SHA, `permissions: contents: read`, Node from `.nvmrc`, tools via `package.json` + lock + `npm ci`.
  Dependabot for `github-actions` and `npm`. Ruleset on `main`: both checks required (source GitHub Actions), branches
  up to date, no bypass; changes only via PR with squash-merge. Verified with a test PR: red on a typo and on a Markdown
  error, merge blocked.
- `barbro`: `LICENSE` AGPL-3.0; private vulnerability reporting enabled. Both repos made public.
- Hetzner account registered; CX23 available. DeepL API Developer key created (regenerate after the in-chat tests).
- M0 sandbox (outside the repos, to be discarded): `nest new` (ESM, Vitest), Express → Fastify, `GET /health` and
  `POST /abv` test-first, outside-in, Zod validation through `StandardSchemaValidationPipe` as `APP_PIPE`.
- Cheat sheet (Claude
  Doc): [NestJS 12 + Fastify + Vitest, outside-in TDD](https://claude.ai/code/artifact/508f0c8f-e022-4509-bad7-1ded9016eb25).
- ADR-0006 amended (2026-09-28): Node.js 26 LTS instead of 24; NestJS built-in Standard Schema validation instead of
  `nestjs-zod`; license open item resolved.
- `barbro` scaffold, merged via PR:
  - Root: `pnpm-workspace.yaml` (`apps/*`, `packages/*`, catalog, supply-chain settings), root `package.json` with
      workspace tooling only, `.nvmrc`, `.editorconfig`, `.gitignore`, `biome.json`.
  - `apps/api` (`@barbro/api`): from `nest new`, cleaned (oxlint, prettier, samples, `test/` removed).
  - `src/app.setup.ts`: adapter factory plus `configureApp` (global prefix `api`), shared by `main.ts` and
      `createTestApp`.
  - `src/config/env.ts`: `parseEnv(source)` with Zod and unit tests.
  - `src/health/`: `HealthModule` and three HTTP tests: 200 with `{ status: "ok" }`, trailing slash, 404 without the
      prefix.
  - `main.ts`: config first, readable error and `exitCode = 1` on invalid env, `listen(port, host)`.

## Decisions

- ADR-0001 — Bar Assistant data (MIT) as the seed: all 663 recipes, status imported/reviewed, steps and descriptions
  rewritten by an LLM, no images.
- ADR-0002 — the LLM only for FR-3b: a sommelier over a deterministic list; FR-3a without the LLM.
- ADR-0003 — English data, i18n table; free text via DeepL (successor — Azure F0); no streaming; LLM behind an adapter,
  model by eval, fallback GPT-6 Luna.
- ADR-0004 — catalog tree for navigation, matching via `satisfies_parent` + a substitution table; default pantry.
- ADR-0005 — sign-in only via Google OIDC, server-side sessions in an httpOnly cookie, `user_identity` separate.
- ADR-0006 (amended 2026-09-28) — TypeScript, Node 26 LTS, NestJS/Fastify, Vite + React SPA, SQLite + Drizzle, Hetzner
  CX23 + Docker Compose + Caddy, GitHub Actions + GHCR with pull deploy, Sentry + UptimeRobot, Litestream → R2,
  Cloudflare DNS; monorepo `barbro`; Zod via Nest's built-in Standard Schema support (no `nestjs-zod`).
- Licenses: `barbro` — `AGPL-3.0-or-later` (SPDX id in every `package.json`), `barbro-docs` — CC BY 4.0. Accepted risk:
  AGPL reduces reuse of the code by people reading the repo as a sample; changeable while SS is the only author.
- Documentation norms: American English (en-US); markdownlint-cli2 with MD013 off; cspell en-US + `@cspell/dict-uk-ua`.
- CI supply chain: third-party actions pinned by full commit SHA with the exact version in a comment; `GITHUB_TOKEN`
  read-only by default; Dependabot in `barbro-docs` (only actions and two npm tools; no third-party app with write
  access), Renovate in `barbro` (pnpm workspaces and catalogs, versions in Dockerfile URLs via regex managers,
  automerge) — set up in M0.
- Model and effort: routine config review — Medium (in `methodology.md`).
- Program level (in `methodology.md`): repos public after phase 2 closes, repo language English, public repo standard.
- TypeScript 6.x across the monorepo via the pnpm catalog. `nest build` needs the TypeScript programmatic API, which TS
  7.0 lacks (planned for 7.1). Renovate must hold `typescript` below 7 until Nest supports it.
- Node version: `.nvmrc` with an exact version is the single source for local and CI. Locally it is read by `fnm` with
  `--use-on-cd`, which replaced Homebrew `node`; in CI by the setup action. The Docker base image is pinned separately
  and updated by Renovate.
- pnpm workspace rules:
  - The root `package.json` holds workspace tooling only; every package declares everything it imports (no phantom
      dependencies).
  - Package names under the `@barbro/` scope; every package `"private": true`.
  - `engines.node` `^26.0.0` + `engineStrict: true`.
  - Shared versions in `catalog:` (`typescript`, `vitest`, `@vitest/coverage-v8`, `@types/node`).
  - `minimumReleaseAge: 1440` set explicitly.
  - `autoInstallPeers: false`, because pnpm installed optional peers contrary to its docs and pulled
      `@nestjs/platform-express` in (pnpm issue #11155); with Express absent, a Nest app created without the Fastify
      adapter now fails fast.
  - Dependency build scripts allowed only via `pnpm approve-builds`.
  - `packages/shared` deferred until the first shared schema.
- Biome configuration:
  - `@biomejs/biome` pinned exactly; one root `biome.json` with `$schema` from `node_modules`.
  - `linter.rules.preset: "recommended"` (`recommended: true` is deprecated since 2.5).
  - `vcs.useIgnoreFile`, `formatter.useEditorconfig`.
  - Override for `apps/api/**`: `useImportType` off and `unsafeParameterDecoratorsEnabled: true`. In `apps/api`, TS
      error TS1272 (`isolatedModules` + `emitDecoratorMetadata`) enforces `import type` in decorated signatures, and
      HTTP tests catch DI tokens imported as types.
  - Scripts: `check` (read-only), `fix` (`--write`); CI will use `biome ci`.
- Code style: `.editorconfig` is the single source for indentation (2 spaces, one `[*]` section, max line length 120);
  double quotes and semicolons (Biome defaults).
- Vitest: explicit imports from `vitest`, `globals: false`, no `vitest/globals` in `tsconfig` types; no
  `--passWithNoTests`.
- API code conventions (easily reversible, no ADR):
  - NestJS 12 ESM project layout.
  - Global prefix `api` set in Nest, not stripped by Caddy.
  - Prefix and Fastify router options (`ignoreTrailingSlash: true`) live only in `app.setup.ts`, shared by `main.ts`
      and tests.
  - Global pipes, guards, interceptors and filters registered as `APP_*` providers in modules, not via
      `app.useGlobal*()`.
  - HTTP tests colocated as `*.spec.ts`, run in-process via `inject` (no supertest).
  - One feature — one Nest module; `AppModule` only imports feature modules.
  - TDD outside-in: HTTP test first, then service unit tests; thin controllers are not unit-tested.
  - Units in field names (`volumeMl`, `abvPercent`).
- Environment config:
  - `process.env` is read only in `main.ts` and passed to `parseEnv(source)` in `src/config/env.ts`: a Zod schema,
      output in camelCase.
  - `PORT`: integer 1–65535, default 3000. `HOST`: IPv4 only, default `127.0.0.1`; the container sets `HOST=0.0.0.0`.
  - Invalid env: `z.prettifyError` under a header in stderr, `process.exitCode = 1`, no app created.
  - Secrets will join the same schema; exposing config to DI is decided when the first adapter needs a key.

## Stack and tools

- **Runtime and tooling:** Node 26 LTS (`fnm` + `.nvmrc` locally), TypeScript 6, pnpm 12.6 (Homebrew; version pinned via
  `packageManager`).
- **API:** NestJS 12 (ESM) on Fastify 5, Zod 4 through Nest's built-in Standard Schema validation, OpenAPI.
- **Web client:** Vite, React, TanStack Query, Tailwind, shadcn/ui, PWA.
- **Data:** SQLite (better-sqlite3 13 on Node-API), Drizzle, Litestream.
- **Infrastructure:** Docker Compose, Caddy; GitHub Actions, GHCR; Sentry, UptimeRobot; Cloudflare DNS/proxy/R2.
- **Quality and testing:** Biome 2.5; Vitest 4 (handles Nest decorator metadata out of the box, no SWC plugin);
  fast-check for property-based tests; Playwright; gitleaks; Renovate; Bruno for the API.
- **Docs repo:** markdownlint-cli2, cspell, Dependabot. Diagrams — Mermaid in Markdown.
- **Adapters:** `Translator` (DeepL → Azure), `Sommelier` (provider SDK) — DI tokens as abstract classes.

## Open questions

- `minimumReleaseAge`: confirm `pnpm config get minimumReleaseAge` returns 1440 (it returned `undefined` before the
  setting was added). Verify against pnpm docs the claim that the built-in default runs in a "loose mode" (
  auto-excluding immature versions) and an explicit value switches to strict.
- Response serialization and OpenAPI generation — decide at the first endpoint with a real contract.
- Health endpoint: add the image version or commit SHA at the pull deploy step, to see what runs on the server.
- Renovate rules to write: hold `typescript` below 7; `.nvmrc` and the Docker base image in sync.
- Dependabot in `barbro-docs`: confirm both ecosystems run without errors and the first PR has a correct commit title (
  `ci(deps): …`, `chore(deps-dev): …`).
- `adr/README.md`: status of ADR-0006 → "accepted, amended".
- OIDC theory: proposed to move it to the start of the auth milestone (no auth in M0); confirm.
- LLM model — by the eval set (~20 EN cases, with injection); candidates GPT-6 Luna, Gemini 3.5 Flash-Lite, Claude Haiku
  4.5.
- Azure Translator F0 — test on the same phrases before the i18n milestone (needs a card).
- Domain — blocks the Caddy + TLS step of M0.
- 18+ for an alcohol site (Legal, before phase 4). Privacy policy with the list of processors. "Source" link in the UI
  footer (AGPL section 13) — phase 4 checklist.
- Protecting cocktail names in DeepL (`tag_handling`) — verify at implementation.
- Google Gemini prices — verify on the official page before the eval.
- Organizational: Claude Code in a clone of the repo for reviews — optional.

## Learned in this product

- The expert council as a format: options with trade-offs, choice by the architect, risks in the ADR — practiced on six
  decisions.
- Measurements instead of assumptions: source coverage, MT quality, prices, and in CI — reproducing a config locally
  before judging it; twice "it seems" did not match the facts.
- Taxonomy ≠ interchangeability (catalog model); denial of wallet as the main threat of a free AI service; validating
  the LLM's structured output against the candidate list.
- Reviewing our own package found three blockers — contradictions between documents are visible only when they are read
  together.
- Writing documents for a public repo from the start is cheaper than translating them afterwards.
- CI supply chain: tags are mutable, SHAs are not; pinning actions while installing tools unpinned with `npm install -g`
  moves the same risk one level down. `--fix` in CI turns a check into a no-op for fixable rules. A required check is
  only real after a deliberately failing PR proves it blocks.
- GitHub specifics: workflows only in `.github/workflows/`; required checks are named by the job `name:`, added in
  Rulesets manually, source pinned to GitHub Actions.
- NestJS modules, controllers, providers and DI tokens — understood and practiced; tokens as abstract classes for
  adapters — theory only.
- TypeScript under ESM: `.js` in relative imports, `import type` for types in decorated signatures but never for DI
  tokens, structural typing — practiced.
- Outside-in TDD (double loop, red for the right reason, fake it, triangulation, tests on behavior) with Vitest and
  Fastify `inject` — practiced. A test that is green from its first run must be proven able to fail by breaking the code
  on purpose — practiced. Property-based testing — theory only.
- Test app parity: configuration shared between `main.ts` and tests (adapter factory, prefix, `APP_*` providers) —
  learned from a failing case, applied in `barbro`.
- Validation: pipes, Zod schema as the single source of rules and type, `NaN` → `null` as a silent failure mode —
  practiced.
- npm supply chain: installing the look-alike unscoped `biome` instead of `@biomejs/biome` pulled 2016-era dependencies;
  pnpm's blocking of dependency build scripts stopped it. Install by the exact name from official docs; never approve
  builds to make an error go away — practiced. `minimumReleaseAge` — understood.
- pnpm workspaces: workspace vs catalog, `pnpm -r`, phantom dependencies (work locally by walking up to the root
  `node_modules`, break in an isolated Docker install), peer and optional peer dependencies, `autoInstallPeers`,
  `pnpm why`, lockfile `settings` — practiced.
- Node version management: `.nvmrc` selects the version (`fnm`, CI), `engines` + `engineStrict` checks it — practiced.
- Biome: one tool for format, lint and import sorting; `check` / `check --write` / `ci`; rule groups, `overrides`,
  `preset`, `vcs`, `.editorconfig`, local JSON schema. Config field names must be checked against the installed
  version (a deprecated field was caught by Biome's own warning) — practiced.
- Environment config: `process.env` holds only strings; `z.coerce.number()` accepts anything `Number()` does (`""` → 0,
  hex, exponent), so range rules do the real work; fail fast at startup; pure function with the source injected;
  `process.exitCode` instead of `process.exit()` — practiced.
- Vitest globals vs explicit imports: `tsconfig` `types` apply to production code too — understood.
- Learning format:
  - A batch of theory up front did not work; hands-on steps with each concept introduced when the code needs it did.
  - This chat repeatedly relied on terms and config options not yet introduced; every new concept must be explained
      before it is used.
- Not yet: the event loop and synchronous better-sqlite3, OIDC flow, Docker multi-stage builds and `pnpm deploy`.

## Next step

1. New chat "Phase 3, M0: Skeleton — web, README, CI" on Opus 5.5 or Fable 5.1, High.
2. SS scaffolds `apps/web` (`@barbro/web`, private): Vite + React + TS, dev proxy for `/api` to the API, a minimal page;
   `typecheck`, `test`, `build` scripts; Biome `useImportType` stays on there.
3. Claude drafts the `barbro` README (what it is, link to `barbro-docs`, requirements: Node via `.nvmrc`, pnpm;
   commands; license).
4. CI for `barbro`: `biome ci`, `typecheck`, `test`, `build`, gitleaks; SHA-pinned actions, read-only token; ruleset on
   `main` with required checks proven by a deliberately failing PR; Renovate (pnpm catalogs, hold TypeScript < 7,
   `.nvmrc` and Docker base image).
5. Then the API image in GHCR (Node 26 base, non-root, `HOST=0.0.0.0`), server with firewall and pull deploy, Caddy +
   TLS via Cloudflare. M0 criterion — an empty application reachable over HTTPS, deployed from `main` hands-off.
6. In parallel, SS: domain (blocks the TLS step), Azure F0 test, the LLM eval set; confirm Dependabot; update the
   ADR-0006 status in `adr/README.md`.
