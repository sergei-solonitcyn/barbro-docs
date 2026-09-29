# STATE — BarBro

Updated: 2026-09-29

## Phase and milestone

Phase 3 Implementation, milestone M0 "Skeleton" — about 60%. The `barbro` monorepo has `apps/api` (NestJS 12 ESM on
Fastify, `GET /api/health`) and `apps/web` (Vite + React SPA with a dev proxy to the API), both built test-first; a
README; CI with five required checks; self-hosted Renovate with automerge. Remaining in M0: the API image in GHCR, the
server with firewall and pull deploy, Caddy + TLS. Both repos are public.

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
- `apps/web` (`@barbro/web`), merged via PR: from the `create-vite` `react-ts` template, cleaned (oxlint, sample assets
  and CSS, the template's own lockfile removed); `App` renders the `h1` "BarBro", covered by a component test that was
  red for the right reason and proven able to fail; dev proxy `/api` → `127.0.0.1:3000` checked with `curl` against a
  running and a stopped API.
- `barbro` fixes: `zod` moved from `devDependencies` to `dependencies` of `@barbro/api` (it is imported at runtime and
  would be missing from a production install); lint-staged pattern fixed (`.tsx` was not covered; Biome 2.5.14 does not
  process YAML and failed YAML-only commits); stray `apps/web/pnpm-lock.yaml` removed.
- `barbro` README: what it is, link to `barbro-docs`, requirements, commands, API configuration, checks, contributing,
  security, license.
- CI for `barbro` (`.github/workflows/ci.yaml`): jobs `lint`, `typecheck`, `test`, `build`, `secrets` — all required in
  the ruleset on `main`, each proven by a deliberately failing PR.
- Renovate for `barbro`: own GitHub App `barbro-renovate` installed only on `barbro`; `.github/workflows/renovate.yaml`
  on a schedule; `renovate.json`. Verified end to end: pin PR with the root lockfile updated, Dependency Dashboard
  lists npm (with catalog), nvm, github-actions and both regex dependencies; the Renovate image was bumped with tag and
  digest.

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
  access), Renovate in `barbro`.
- CI for `barbro`:
  - One workflow `ci.yaml`, one job per gate (`lint`, `typecheck`, `test`, `build`, `secrets`), so a red PR names the
      failed gate; job ids are the check names in the ruleset, chosen before adding them.
  - `pnpm/setup` v3 installs pnpm (from `packageManager`, signature-verified) and Node (from `.nvmrc`), with cache and
      `require-lockfile`; replaces `setup-node` + `pnpm/action-setup`.
  - `pnpm exec biome ci` (`run` steps do not see `node_modules/.bin`); `pnpm -r` without `--parallel`, which would
      ignore the topological order.
  - Every job: `timeout-minutes`, `persist-credentials: false`; workflow-level `permissions: contents: read`.
  - gitleaks: the MIT binary, not `gitleaks-action` (proprietary license since v2). Version (with the `v` tag prefix)
      and the tarball SHA-256 pinned in the workflow, archive and binary in `$RUNNER_TEMP`,
      `gitleaks git --redact --verbose` over the full history (`fetch-depth: 0`).
- Renovate in `barbro` — self-hosted, not the Mend-hosted app, to keep the "no third-party app with write access" rule:
  - Own private GitHub App, installed only on `barbro`; permissions as listed in the Renovate docs. Client ID in a
      repository variable, private key in a secret; a one-hour installation token via `actions/create-github-app-token`.
  - `renovatebot/github-action` on cron `17 */4 * * *` plus `workflow_dispatch` with a `log_level` input; `concurrency`
      group; `RENOVATE_PLATFORM_COMMIT: enabled` (verified commits); `RENOVATE_DOCKER_MAX_PAGES: 10`.
  - The Renovate image pinned as `docker.io/renovate/renovate:<tag>@sha256:<digest>` (never the action's floating
      default `'44'`); taken from Docker Hub, because Renovate gets Docker release timestamps only there.
  - `renovate.json`: `config:best-practices`; `minimumReleaseAge: "3 days"` for every datasource (the preset covers
      npm only); `typescript` held below 7; automerge for `minor`, `patch`, `pin`, `digest`, majors by hand; the
      repository setting "Allow auto-merge" on.
  - Custom regex managers: gitleaks (`github-release-attachments`, recomputes the SHA-256 from the release checksums)
      and the Renovate image (`docker`, tag and digest).
- Model and effort: routine config review — Medium (in `methodology.md`).
- Program level (in `methodology.md`): repos public after phase 2 closes, repo language English, public repo standard.
- TypeScript 6.x across the monorepo via the pnpm catalog. `nest build` needs the TypeScript programmatic API, which TS
  7.0 lacks (planned for 7.1). Renovate holds `typescript` below 7 until Nest supports it.
- Node version: `.nvmrc` with an exact version is the single source for local and CI. Locally it is read by `fnm` with
  `--use-on-cd`, which replaced Homebrew `node`; in CI by the setup action. The Docker base image is pinned separately
  and updated by Renovate.
- pnpm workspace rules:
  - The root `package.json` holds workspace tooling only; every package declares everything it imports (no phantom
      dependencies).
  - Package names under the `@barbro/` scope; every package `"private": true`.
  - `engines.node` `^26.0.0` + `engineStrict: true`.
  - Shared versions in `catalog:` (`typescript`, `vitest`, `@vitest/coverage-v8`, `@types/node`, `vite`, `zod`).
  - One lockfile, at the root; `.gitignore` ignores `pnpm-lock.yaml` everywhere except the root.
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
  - Scripts: `check` (read-only), `fix` (`--write`); CI uses `biome ci`.
  - React rules in `apps/web` come from Biome's `react` domain, auto-enabled by `react` in the nearest
      `package.json` (verified on 2.5.14); no override needed, `useImportType` stays on.
- Git hooks: husky + lint-staged, pre-commit runs `biome check --write --no-errors-on-unmatched` on staged
  `js,jsx,ts,tsx,json,jsonc,css`. GUI git clients do not load the shell `PATH`, so each machine has
  `~/.config/husky/init.sh` (Homebrew, `fnm env`, `fnm use`) outside the repo.
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
- Web code conventions (easily reversible, no ADR):
  - Solution-style `tsconfig` (app + node projects); `typecheck` is `tsc -b`, because `tsc --noEmit` on the root
      config checks no files and exits 0 (measured). `build` is `vite build` only; Vite does not type-check.
  - `moduleResolution: "bundler"`, relative imports with the real `.ts`/`.tsx` extension; `verbatimModuleSyntax` and
      `erasableSyntaxOnly` in `apps/web` only (the API relies on parameter properties).
  - Components as named exports; `main.tsx` throws a readable error if `#root` is missing.
  - Component tests colocated as `*.spec.tsx`: Vitest with `jsdom`, Testing Library, queries by role first; a setup
      file calls `cleanup` in `afterEach`, since auto-cleanup needs Vitest globals (reproduced).
  - Dev proxy `/api` → `http://127.0.0.1:3000`, prefix kept, so dev has the same single origin as production.
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
- **Web client:** Vite 8, React 19, TanStack Query, Tailwind, shadcn/ui, PWA; tests with Vitest, jsdom and Testing
  Library.
- **Data:** SQLite (better-sqlite3 13 on Node-API), Drizzle, Litestream.
- **Infrastructure:** Docker Compose, Caddy; GitHub Actions, GHCR; Sentry, UptimeRobot; Cloudflare DNS/proxy/R2.
- **Quality and testing:** Biome 2.5; Vitest 4 (handles Nest decorator metadata out of the box, no SWC plugin);
  fast-check for property-based tests; Playwright; gitleaks (binary in CI); Renovate (self-hosted, own GitHub App);
  husky + lint-staged; Bruno for the API.
- **Docs repo:** markdownlint-cli2, cspell, Dependabot. Diagrams — Mermaid in Markdown.
- **Adapters:** `Translator` (DeepL → Azure), `Sommelier` (provider SDK) — DI tokens as abstract classes.

## Open questions

- `minimumReleaseAge`: confirm `pnpm config get minimumReleaseAge` returns 1440 (it returned `undefined` before the
  setting was added). Verify against pnpm docs the claim that the built-in default runs in a "loose mode" (
  auto-excluding immature versions) and an explicit value switches to strict.
- Response serialization and OpenAPI generation — decide at the first endpoint with a real contract.
- Health endpoint: add the image version or commit SHA at the pull deploy step, to see what runs on the server.
- Renovate: keep `.nvmrc` and the Docker base image in sync — write the rule with the Dockerfile.
- Renovate for gitleaks: the digest path is verified only by reading the datasource code; confirm on the first gitleaks
  release PR (version and SHA-256 change together, `secrets` green).
- Vitest 5: major PR open from Renovate — review the changes and compatibility with Vite 8 before merging.
- husky is flagged as abandoned by Renovate (last release 2024-11) — keep while it works.
- Docker Hub limits: anonymous pulls of the Renovate image and anonymous tag pagination (max 10 pages) — watch the
  Renovate log.
- actionlint as part of the `lint` job — optional; it caught `workflow_dispatch: true` in review.
- How the built SPA reaches Caddy (a Caddy image with `dist/` inside, or a volume) — decide with the images.
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
- React basics: `index.html` → `main.tsx` → `createRoot().render()`; a component is a function React calls; JSX
  compiles to `jsx()` calls, lowercase names are DOM tags (TS reports unknown ones, TS2339); `StrictMode` renders twice
  in dev — understood. Component tests with Testing Library (`render`, `screen`, `getBy`/`queryBy`/`findBy`, roles and
  accessible names; why `getByRole` beats `getByText`) — practiced. Props, state, effects — not yet (start of M1).
- Vite: dev server without bundling vs Rolldown build, no type-checking; bundler resolution vs Node ESM (`.tsx` vs
  `.js` in specifiers); `verbatimModuleSyntax`, `erasableSyntaxOnly` — understood. `tsc --noEmit` silently checking
  nothing on a solution-style config — measured.
- Git hooks: husky runs hooks with `sh`, prepends `node_modules/.bin`, sources `~/.config/husky/init.sh`; GUI clients
  lack the shell `PATH` — practiced.
- GitHub Actions: parallel jobs as separate checks; `run` steps do not see `node_modules/.bin`;
  `pnpm -r --parallel` breaks topological order; `${{ }}` in `run` is a template-injection risk, in `env` it is not;
  `workflow_dispatch` inputs; `concurrency`; `schedule` in UTC and disabled after 60 inactive days in public repos;
  `GITHUB_TOKEN`-created PRs do not trigger workflows freely — practiced.
- Secret scanning: gitleaks scans history, so a PR with a leaked secret cannot be fixed, only closed; GitHub keeps
  `refs/pull/<n>/head`; `--redact` because public logs republish findings — practiced.
- Supply chain, next level: pin downloaded binaries by a SHA-256 kept in the repo (a checksum file from the same release
  protects only against corruption); Docker images by digest (`name:tag@sha256:…`, where `@` separates the digest); a
  git commit SHA is not an image digest; floating image defaults in actions — practiced.
- Renovate: global vs repository config; presets and what `best-practices` adds; `minimumReleaseAge` per datasource,
  `timestamp-required` + `internalChecksFilter: strict` → updates silently pending when a datasource has no timestamps;
  Docker timestamps only from Docker Hub, whose anonymous API stops at page 10; a stray nested lockfile changed the
  lockfile directory and the Node version Renovate used; custom regex managers with `currentDigest`; the config
  validator does not catch patterns that match nothing — verify in the Dependency Dashboard — practiced.
- GitHub Apps: app vs installation, App ID vs Client ID, installation tokens; 401 (bad key) vs 404 (not installed) —
  practiced.
- `.gitignore`: leading whitespace is significant, `!` must be the first character — verified with `git check-ignore`.
- Reading the tool's own source (Renovate, husky, the Vite template, action `action.yml` at the pinned SHA) settled
  questions that docs, README examples and memory got wrong.
- Learning format:
  - A batch of theory up front did not work; hands-on steps with each concept introduced when the code needs it did.
  - Every new concept must be explained before it is used.
  - Every instruction must say where it goes: which repo, which file, which settings page.
- Not yet: the event loop and synchronous better-sqlite3, OIDC flow, Docker multi-stage builds and `pnpm deploy`,
  TanStack Query.

## Next step

1. New chat "Phase 3, M0: Skeleton — image and deploy" on Opus 5.5 or Fable 5.1, High.
2. SS: remove the leading spaces from the two `pnpm-lock.yaml` lines in `.gitignore` and check with
   `git check-ignore -v apps/web/pnpm-lock.yaml pnpm-lock.yaml`.
3. API image: Docker multi-stage build and `pnpm deploy` (theory with the first Dockerfile), Node 26 base pinned by
   digest, non-root, `HOST=0.0.0.0`; Renovate rule keeping the base image in sync with `.nvmrc`; build and push to GHCR
   from `main` in CI, with `packages: write` only in that job; the image version or commit SHA in the health endpoint.
4. How the SPA is served by Caddy; then the server: Hetzner CX23, firewall (22 from SS's IP, 80/443 from Cloudflare
   ranges), pull deploy by digest, Caddy + TLS via Cloudflare. M0 criterion — an empty application reachable over HTTPS,
   deployed from `main` hands-off.
5. In parallel, SS: domain (blocks the TLS step), Azure F0 test, the LLM eval set; confirm Dependabot; update the
   ADR-0006 status in `adr/README.md`.
