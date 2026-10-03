# STATE — BarBro

Updated: 2026-10-03

## Phase and milestone

Phase 3 Implementation, milestone M0 "Skeleton" — about 85%. The `barbro` monorepo has `apps/api` (NestJS 12 ESM on
Fastify, `GET /api/health` with the build revision) and `apps/web` (Vite + React SPA with a dev proxy to the API), both
built test-first; a README; CI with seven required checks; self-hosted Renovate with automerge. CI builds and
smoke-tests two images on every PR through a matrix — `barbro-api` and `barbro-web` (Caddy serving the SPA) — and
publishes both to GHCR from `main`. Remaining in M0: the server with firewall and pull deploy, the edge Caddy + TLS.
Both repos are public.

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
- `.gitignore`: lockfile patterns fixed (`**/pnpm-lock.yaml`, `!/pnpm-lock.yaml`); checked with
  `git check-ignore -v --no-index`, because without `--no-index` tracked files are skipped silently.
- API image `apps/api/Dockerfile`, merged via PR:
  - Build stage `node:26.10.0-trixie-slim` pinned by digest; pnpm installed from `packageManager`; `pnpm fetch` →
    `install --offline --frozen-lockfile` → `build` → `pnpm deploy --prod /out`.
  - Runtime stage `gcr.io/distroless/nodejs26-debian13:nonroot` pinned by digest; `USER 65532:65532`, `HOST=0.0.0.0`;
    application files owned by root.
  - Allowlist `.dockerignore` (`.env*` excluded last); `tsconfig.build.json` excludes `src/testing` and sets
    `declaration: false`.
  - Measured locally: Node 26.10.0 in both stages; `docker stop` took 0.13 s with exit code 0 after
    `app.enableShutdownHooks()` (exit code 137, SIGKILL, before it).
- Build revision in `GET /api/health` (`{ status, revision }`), test-first: `APP_REVISION` in `parseEnv` (default `dev`,
  an empty value fails); `ConfigModule.forRoot(env)` provides `APP_CONFIG`; `AppModule.forRoot(env)` is used by both
  `main.ts` and `createTestApp(source)`; `ARG`/`ENV APP_REVISION` are the last runtime layers. An image built without
  the build arg exits with code 1 and a readable error.
- CI image pipeline for the API:
  - Job `image`: Node version sync check, Buildx build for `linux/amd64` with the GitHub Actions cache, a smoke script
    (Node major vs `.nvmrc`, health, revision equals the commit). Required in the ruleset, proven by a PR with a broken
    `CMD`.
  - Job `publish` on push to `main` after all checks: push `ghcr.io/sergei-solonitcyn/barbro-api:sha-<commit>`, pull
    by digest, smoke-test the pushed image, digest in the job summary.
  - The GHCR package is public. Anonymous pull verified through the registry API: an index with `linux/amd64` plus a
    provenance attestation; user, entrypoint, env and labels as expected.
- Renovate: a `Node.js` group for `.nvmrc` and the Dockerfile, and distroless digest updates without the age check (see
  Decisions). Config validated with Renovate 44.119.1, the version of the self-hosted runner.
- Web image, merged via PR:
  - `apps/web/Caddyfile`: global `admin off` and `persist_config off`; site `:8080` (HTTP only), `root * /srv`; three
    `handle` blocks — `/revision` responds with `{env.APP_REVISION}`; `/assets/*` with
    `Cache-Control: public, max-age=31536000, immutable` only on 2xx (`match status 2xx`), no fallback, so a missing
    chunk is a 404; catch-all with `Cache-Control: no-cache` and `try_files {path} /index.html`. Formatted with
    `caddy fmt` (tabs).
  - `apps/web/Dockerfile`: build stage identical to the API's up to `build`, with `--filter @barbro/web...` and no
    `pnpm deploy`; runtime `caddy:2.11.4-alpine` pinned by index digest; `setcap -r /usr/bin/caddy`; Caddyfile copied
    from the build context, `dist` from the build stage, both owned by root; `caddy fmt` + `caddy validate` at build;
    `USER 65532:65532`; `EXPOSE 8080`; explicit `CMD`; `ARG APP_REVISION` + `RUN test -n "$APP_REVISION"` +
    `ENV APP_REVISION` as the last layers, so a build without the arg fails.
  - `.dockerignore` allowlist extended to `apps/web` (its `dist`, `node_modules`, `coverage` excluded).
  - `.github/scripts/smoke-web-image.sh`: image user `65532:65532`; run with `--read-only`,
    `--tmpfs /data:uid=65532,gid=65532,mode=0700`, `--cap-drop ALL`, `--security-opt no-new-privileges`; `/` is
    `text/html` + `no-cache`; a deep route returns `index.html`; the asset referenced by `index.html` is 200 +
    `immutable`; a missing asset is 404 without `Cache-Control`; `/revision` equals the commit; no `"level":"error"` in
    the logs. `CONTAINER_CLI` overrides `docker` for local runs with podman. Green locally; red on a wrong user,
    revision, missing `match status 2xx` and read-only `/data` (measured by Claude against a local Caddy).
  - `.github/scripts/smoke-image.sh` renamed to `smoke-api-image.sh`.
  - Measured: Caddy 2.11.4 and 2.11.6 behave the same for this config; Caddy stops on SIGTERM with exit code 0 within
    milliseconds (no PID 1 issue); with the file capability left on the binary, `--cap-drop ALL` blocks `execve`
    (`setpriv` exit 126; podman/crun "exec container process `/usr/bin/caddy`: Operation not permitted", exit 1).
- CI matrix for both images, merged via PR:
  - `image` and `publish` run with `strategy.matrix.app: [api, web]`, `fail-fast: false`, `APP` in job-level `env`;
    Buildx cache per image (`scope=api`, `scope=web`); the Node sync check runs per `apps/$APP/Dockerfile`.
  - Ruleset: `image` replaced by `image (api)` and `image (web)` (seven required checks). `image (web)` was proven red
    in CI by a real defect: Docker mounted `--tmpfs /data` not writable for UID 65532 (podman's default was), caught
    only by the "no errors in logs" check.
  - `publish` pushes `ghcr.io/sergei-solonitcyn/barbro-web:sha-<commit>` next to `barbro-api`, each digest in its own
    job summary; the `barbro-web` package made public.

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
  - `renovatebot/github-action` on cron `17 */4 * * *` plus `workflow_dispatch` with a `log_level` input;
    `concurrency` group; `RENOVATE_PLATFORM_COMMIT: enabled` (verified commits); `RENOVATE_DOCKER_MAX_PAGES: 10`.
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
  `--use-on-cd`, which replaced Homebrew `node`; in CI by the setup action. The build stage of both Dockerfiles uses the
  same version (Renovate group plus a CI check per Dockerfile); the distroless runtime is checked by Node major only.
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
  double quotes and semicolons (Biome defaults). Caddyfiles use tabs, as `caddy fmt` writes them.
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
  - Secrets will join the same schema.
  - Config in DI: `ConfigModule.forRoot(env)` is a global dynamic module with a `useValue` provider under the
    `APP_CONFIG` symbol token; consumers use `@Inject(APP_CONFIG)` with `import type { Env }`. Abstract-class tokens
    stay for adapters with behavior; a class + interface merge for config is rejected by Biome
    (`noUnsafeDeclarationMerging`).
  - `APP_REVISION`: non-empty string, default `dev`; the image sets it from the build arg.
- API container image (easily reversible, no ADR):
  - Runtime base distroless `nodejs26-debian13:nonroot`, chosen over `node:26-trixie-slim` (shell and more packages)
    and Docker Hardened Images (authenticated pulls in CI and Renovate). Build stage `node:<.nvmrc>-trixie-slim`,
    the same Debian 13 glibc. Both `FROM` pinned by index digest.
  - `pnpm deploy` on the non-legacy path: `injectWorkspacePackages: true`; `files: ["dist"]` in `apps/api` decides
    what is deployed. `--filter @barbro/api...` for install and build, so future workspace packages are included.
  - Node is PID 1 and gets no default SIGTERM action, so `app.enableShutdownHooks()` is required. Accepted risk: a
    SIGTERM during bootstrap, before that call, is lost and the container waits for SIGKILL; nothing is in flight
    then.
  - Registry platform `linux/amd64` only (the server is x86); local builds are native arm64.
  - No secrets through `--build-arg`: provenance `mode=max`, the default for public repos, records build args in the
    attestation. Use BuildKit `secrets` when needed.
- Serving the SPA (easily reversible, no ADR; one-role decision, platform engineer):
  - A separate `barbro-web` image — Caddy as a static server on `:8080` with `dist` baked in — behind an edge Caddy:
    the official image pinned by digest with its Caddyfile from `infra/`, owning TLS, security headers and routing
    (`/api/*` → `api`, the rest → `web`). Three containers on the server: `edge`, `web`, `api`.
  - Why: what changes often (SPA, API) ships as immutable images with one CI pattern (build, smoke, publish, digest);
    what changes rarely and holds TLS state (edge) lives apart, and an SPA deploy does not restart it. The edge
    Caddyfile reaches the server the same way `compose.yaml` must anyway.
  - Rejected: an edge image with `dist` inside (every SPA deploy restarts TLS and the `/api` proxy; the CI smoke test
    cannot run the production Caddyfile without the certificate), a volume filled from a separate artifact (a second
    delivery channel with its own integrity check and atomic swap), the API serving static files via
    `@fastify/static` (static traffic through the Node event loop, SPA deploy = API restart).
  - Caching: hashed `/assets/*` are `immutable` on 2xx only; `index.html` and other root files are `no-cache`; a
    missing asset is a 404, never `index.html`.
- Web container image (easily reversible, no ADR):
  - Runtime base official `caddy:<version>-alpine` (Docker Hub, so Renovate's `minimumReleaseAge` works for it).
    Image tags are checked in the registry itself, not in `docker-library/official-images` (2.11.6 was merged there a
    day before it was published).
  - `setcap -r /usr/bin/caddy`: the official image grants `cap_net_bind_service` to the binary, and with
    `cap_drop: ALL` the kernel refuses `execve`. `web` listens on 8080 and needs no capability. Cost: a ~40 MB layer.
  - `admin off` (no runtime config API in an immutable image), `persist_config off` (no `autosave.json` writes).
  - Caddy's TLS module touches `/data` on start even without TLS sites, so a read-only root filesystem needs a tmpfs on
    `/data` with explicit `uid=65532,gid=65532,mode=0700` — Docker and podman mount `--tmpfs` with different defaults.
  - `caddy fmt` + `caddy validate` in the image build. Accepted: an unformatted Caddyfile turns `image (web)` red, not
    `lint`; the Caddy binary is needed only there.
  - `/revision` serves `APP_REVISION`, the web counterpart of `revision` in `/api/health`; an empty value is blocked at
    build by `RUN test -n "$APP_REVISION"`, since Caddy renders an unset placeholder as an empty string silently.
  - Security headers and compression belong to the edge Caddyfile, not to `web`.
- CI image pipeline:
  - `image` runs on every PR and push with `contents: read`; `publish` runs only on push to `main` and is the only job
    with `packages: write`.
  - Both jobs are a matrix over `app: [api, web]` with `fail-fast: false`, so a red leg does not cancel the other and
    each image is its own required check (`image (api)`, `image (web)`). Values reach `run` steps through `env`, never
    `${{ }}`.
  - Buildx cache `type=gha` with a separate `scope` per image; with the default scope the second build overwrites the
    first one's cache index. Caches created by PRs are not readable from `main`.
  - `publish` rebuilds from cache, pushes, then pulls by digest and smoke-tests, so the tested artifact is the one
    deployed. Accepted: `publish` is first exercised after merge (a missing `tags:` was caught that way).
  - `publish` is not a required check: on PRs it is skipped by its `if`, and a skipped job reports success. Its gate
    belongs to the deploy — the server takes a pair of digests only from a commit whose whole `publish` matrix
    succeeded.
  - Smoke scripts per image, `.github/scripts/smoke-<app>-image.sh`, with the production runtime flags.
  - Tags only `sha-<full commit>`, no `latest`; deploy goes by digest.
  - OCI labels `source`, `revision`, `licenses` written by hand instead of `docker/metadata-action`.
  - Public GHCR packages: anonymous pull, no GitHub token on the server.
- Renovate rules added:
  - Group `Node.js`: `matchManagers: ["nvm", "dockerfile"]`, `matchDepNames: ["node"]`; covers the build stage of both
    Dockerfiles. The `image` jobs fail when a Dockerfile build stage differs from `.nvmrc`, so a partial group PR
    cannot automerge.
  - Distroless digest updates: `minimumReleaseAgeBehaviour: "timestamp-optional"`. Its tag is unversioned and gcr.io
    has no release timestamps, so the updates would be held forever. Accepted risk: no cooldown for this base image;
    compensating control — cosign signature verification (open question).
  - Validation uses the runner's version, from the repo root and without a file argument (with one, the file is
    validated as global config): `npx --yes --package renovate@<version> -- renovate-config-validator`.

## Stack and tools

- **Runtime and tooling:** Node 26 LTS (`fnm` + `.nvmrc` locally), TypeScript 6, pnpm 12.6 (Homebrew; version pinned via
  `packageManager`).
- **API:** NestJS 12 (ESM) on Fastify 5, Zod 4 through Nest's built-in Standard Schema validation, OpenAPI.
- **Web client:** Vite 8, React 19, TanStack Query, Tailwind, shadcn/ui, PWA; tests with Vitest, jsdom and Testing
  Library. Served by Caddy 2.11 (`barbro-web` image).
- **Data:** SQLite (better-sqlite3 13 on Node-API), Drizzle, Litestream.
- **Infrastructure:** Docker (multi-stage, distroless and Alpine runtimes, Buildx), Docker Compose, Caddy (edge and
  static); GitHub Actions, GHCR; Sentry, UptimeRobot; Cloudflare DNS/proxy/R2. Locally Docker Desktop on macOS (arm64)
  and podman on one of the machines.
- **Quality and testing:** Biome 2.5; Vitest 4 (handles Nest decorator metadata out of the box, no SWC plugin);
  fast-check for property-based tests; Playwright; gitleaks (binary in CI); Renovate (self-hosted, own GitHub App);
  husky + lint-staged; Bruno for the API; bash + curl image smoke tests.
- **Docs repo:** markdownlint-cli2, cspell, Dependabot. Diagrams — Mermaid in Markdown.
- **Adapters:** `Translator` (DeepL → Azure), `Sommelier` (provider SDK) — DI tokens as abstract classes.

## Open questions

- Pull deploy: how the server learns the pair of digests of one commit, taken only when the whole `publish` matrix
  succeeded (an image of one app may be published without the other). `threat-model.md` says the server pulls by the
  `main` tag, while CI publishes only `sha-<commit>` tags — resolve in the deploy step, then align `threat-model.md`.
- How `infra/` (`compose.yaml`, the edge Caddyfile) reaches the server — decide with the pull deploy.
- `architecture.md`: the Container diagram and table show one Caddy serving the SPA; update to `edge` + `web` + `api`
  (Claude drafts it in the edge step).
- Edge Caddy: the official image's file capability vs `cap_drop: ALL` while binding 80/443 (`cap_add:
  NET_BIND_SERVICE` or high ports with a port mapping); `grace_period` in the Caddyfile aligned with
  `stop_grace_period` in Compose (Caddy's default waits for active requests forever, and LLM calls through `/api` take
  seconds); whether `/data` must persist depends on how TLS is done with Cloudflare.
- Compose on the server: every tmpfs with explicit `uid`, `gid`, `mode` (runtime defaults differ, measured in CI).
- API smoke script: align its run flags with the web one (`--read-only`, `--cap-drop ALL`, `no-new-privileges`) —
  optional.
- Caddy 2.11.6 is in `docker-library/official-images` but not yet on Docker Hub — Renovate picks it up after the
  3-day cooldown.
- `minimumReleaseAge`: confirm `pnpm config get minimumReleaseAge` returns 1440 (it returned `undefined` before the
  setting was added). Verify against pnpm docs the claim that the built-in default runs in a "loose mode"
  (auto-excluding immature versions) and an explicit value switches to strict.
- Response serialization and OpenAPI generation — decide at the first endpoint with a real contract.
- Distroless base image: verify its cosign signature in CI before the build — the compensating control for skipping
  `minimumReleaseAge` on its digests.
- Renovate Dependency Dashboard: confirm distroless is no longer under "Pending Status Checks", and that `.nvmrc` and
  the `node` build stage of both Dockerfiles land in the `Node.js` group on the next Node release. The validator does
  not catch a wrong manager name (`nvmrc` passed on 44.119.1 — measured).
- A Node major upgrade is manual: the distroless image name (`nodejs26-…`) and the Renovate rule's package name change
  with it.
- Docker Desktop stopped containers after about 3 s instead of 10 s with `StopTimeout` unset — cause unknown, irrelevant
  for the server; set `stop_grace_period` explicitly in Compose.
- Renovate for gitleaks: the digest path is verified only by reading the datasource code; confirm on the first gitleaks
  release PR (version and SHA-256 change together, `secrets` green).
- Vitest 5: major PR open from Renovate — review the changes and compatibility with Vite 8 before merging.
- husky is flagged as abandoned by Renovate (last release 2024-11) — keep while it works.
- Docker Hub limits: anonymous pulls of the Renovate image and anonymous tag pagination (max 10 pages) — watch the
  Renovate log.
- actionlint as part of the `lint` job — optional; it caught `workflow_dispatch: true` in review.
- Dependabot in `barbro-docs`: confirm both ecosystems run without errors and the first PR has a correct commit title
  (`ci(deps): …`, `chore(deps-dev): …`).
- `adr/README.md`: status of ADR-0006 → "accepted, amended".
- OIDC theory: proposed to move it to the start of the auth milestone (no auth in M0); confirm.
- LLM model — by the eval set (~20 EN cases, with injection); candidates GPT-6 Luna, Gemini 3.5 Flash-Lite, Claude Haiku
  4.5.
- Azure Translator F0 — test on the same phrases before the i18n milestone (needs a card).
- Domain — blocks the edge Caddy + TLS step of M0.
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
- Docker for a Node monorepo: `pnpm fetch` needs only the lockfile and ignores `--filter`; `pnpm deploy --prod` needs
  `injectWorkspacePackages`, and `files` decides its content; `--filter pkg...`; Corepack is not shipped with Node 25+;
  distroless (no shell, `--entrypoint` to run `node` for checks); index digests cover all platforms — practiced.
- PID 1 signals: no default SIGTERM action, exit code 137 vs 0, Nest's re-raised signal being ignored; diagnose by exit
  codes, not timings; a race between `docker run` and `docker stop` looked like a broken handler — practiced.
- TypeScript config: an unknown top-level key (`declaration` outside `compilerOptions`) is ignored silently — measured
  on TS 6.0.3.
- NestJS dynamic modules (`forRoot`, `DynamicModule`, `global`), `useValue` providers, symbol tokens with `@Inject` —
  practiced. HTTP tests check the configured value; defaults belong to the config unit tests.
- GitHub Actions and containers: Buildx `docker-container` driver (gha cache, `load: true`), path vs Git context,
  job-level `packages: write`, GHCR packages private on first push, provenance attestation shown as `unknown/unknown`
  — practiced.
- curl retries: a reset from the Docker port proxy is not "connection refused", so `--retry-all-errors` — understood.
- Renovate and digests: Docker Hub ages digests by `tag_last_pushed`; other registries and unversioned tags are held
  forever under `timestamp-required`; the validator must match the runner's version — practiced.
- Learning format, additions:
  - SS holds a Docker certification: skip Docker basics; teach the Node- and pnpm-specific parts.
  - A term introduced earlier must be named again in full when reused ("hooks" alone was unclear).
  - Dense prose specs of CI jobs did not work; on SS's request an annotated example of the jobs and the smoke script
    did. When the volume of new material became overwhelming, a ready, tested script plus a short list of SS's own
    actions (run green, run red, bring the output) worked.
- Caddy: Caddyfile structure (global options block vs site block — a missing site block makes every directive a
  "global option"), site address without a host = plain HTTP, `root`, mutually exclusive `handle` blocks sorted by
  matcher specificity, `try_files`, `file_server`, `header` with a response matcher (`match status 2xx`), `respond`,
  placeholders (`{env.X}` at request time renders an unset variable as an empty string), `admin`, `persist_config`, the
  TLS module's storage maintenance even without TLS, `caddy fmt` / `caddy validate` exit codes — practiced.
- SPA serving: client-side routes need the `index.html` fallback; content-hashed assets are cached forever, the HTML
  never; a missing asset must stay a 404, or the browser runs HTML as a script — understood and practiced.
- Linux file capabilities: a binary with a file capability cannot be executed when that capability is outside the
  bounding set (`cap_drop: ALL`) — measured with `setpriv` and podman; fixed by `setcap -r` when the capability is not
  needed.
- Container runtimes differ in defaults: `--tmpfs` writable for a non-root user in podman, not in Docker — caught only
  by a "no errors in logs" check; set ownership and mode explicitly.
- Image references: `registry/namespace/name:tag@digest`; no registry means `docker.io`, no namespace means `library/`;
  a digest belongs to one repository; podman resolves short names via `registries.conf`; a tag can exist in the
  upstream definition before the registry has it — practiced.
- Dockerfile review: a trailing `\` swallows the next instruction (here `USER`, silently, because `caddy validate`
  ignores extra arguments) — verify `Config.User` with `image inspect` and count the build steps — learned from review.
- Bash for smoke tests: under `set -e` + `pipefail` a `grep` with no match aborts the script silently, so `|| true` and
  decide in the next check; `cmd | grep -q` can fail on SIGPIPE under `pipefail`, so capture output first;
  `curl -w '%{http_code}'`, `curl -D -` — understood.
- GitHub Actions matrix: `strategy.matrix`, `fail-fast`, check names `job (value)`, matrix context in job-level `env`,
  a separate gha cache `scope` per build; a required check renamed by a matrix blocks its own PR until the ruleset is
  switched; a job skipped by `if` reports success, so it cannot be a meaningful required check — practiced.
- Not yet: the event loop and synchronous better-sqlite3, OIDC flow, TanStack Query.

## Next step

1. New chat "Phase 3, M0: Skeleton — server and deploy" on Opus 5.5 or Fable 5.1, High.
2. Server: Hetzner CX23 (x86), firewall (22 from SS's IP, 80/443 from Cloudflare ranges), Docker + Compose with three
   services `edge`, `web`, `api` (`stop_grace_period`, read-only root filesystem, `cap_drop: ALL`, tmpfs with explicit
   `uid`/`gid`/`mode`).
3. Pull deploy: how the server finds the digest pair of one commit (only after the whole `publish` matrix succeeded)
   and how `infra/` reaches it; then align `threat-model.md`.
4. Edge Caddy + TLS via Cloudflare; Claude updates the Container diagram in `architecture.md`. M0 criterion — an empty
   application reachable over HTTPS, deployed from `main` hands-off; `/api/health` and `/revision` show the deployed
   revision.
5. In parallel, SS: domain (blocks the TLS step), Azure F0 test, the LLM eval set; confirm Dependabot; update the
   ADR-0006 status in `adr/README.md`; check the Renovate Dependency Dashboard (distroless, `Node.js` group, Caddy).
