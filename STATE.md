# STATE — BarBro

Updated: 2026-10-02

## Phase and milestone

Phase 3 Implementation, milestone M0 "Skeleton" — about 75%. The `barbro` monorepo has `apps/api` (NestJS 12 ESM on
Fastify, `GET /api/health` with the build revision) and `apps/web` (Vite + React SPA with a dev proxy to the API), both
built test-first; a README; CI with six required checks; self-hosted Renovate with automerge. CI builds and smoke-tests
the API image on every PR and publishes it to GHCR from `main` (public package). Remaining in M0: how the SPA is served,
the server with firewall and pull deploy, Caddy + TLS. Both repos are public.

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
  the
  build arg exits with code 1 and a readable error.
- CI image pipeline:
  - Job `image`: Node version sync check, Buildx build for `linux/amd64` with the GitHub Actions cache,
      `.github/scripts/smoke-image.sh` (Node major vs `.nvmrc`, health, revision equals the commit). Required in the
      ruleset, proven by a PR with a broken `CMD`.
  - Job `publish` on push to `main` after all six checks: push `ghcr.io/sergei-solonitcyn/barbro-api:sha-<commit>`,
      pull by digest, smoke-test the pushed image, digest in the job summary.
  - The GHCR package is public. Anonymous pull verified through the
