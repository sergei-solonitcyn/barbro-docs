# STATE — BarBro

Updated: 2026-10-06

## Phase and milestone

Phase 3 Implementation, milestone M1 "Sign-in" — increment 1 of 4 done (about 25%). M0 "Skeleton" is closed: the last
merge deployed hands-off with all four services. The `barbro` monorepo has `apps/api` (NestJS 12 ESM on Fastify; SQLite
through better-sqlite3 + Drizzle, migrations applied at startup; `GET /api/health` reports the build revision and the
database state; JSON logs through nestjs-pino) and `apps/web` (Vite + React SPA), both built test-first; CI with eight
required checks and a schema/migration drift step; self-hosted Renovate with automerge. CI builds and smoke-tests the
`barbro-api` and `barbro-web` images, runs a system smoke test of the whole Compose stack through the edge, and
publishes both images to GHCR from `main`. Production: `https://barbro.dev` — Cloudflare → Cloudflare Tunnel
(`cloudflared`) → edge Caddy → `web` / `api` on a Hetzner CX23 with no inbound HTTP ports; a pull deploy agent
(ADR-0007) deploys every green merge to `main` by digest; the database lives in the named volume `barbro_api-data`. Both
repos are public.

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
- Server bootstrap, merged via PR:
  - `infra/cloud-init.yaml`: user `ss` with two ed25519 keys (one per machine), passwordless sudo, `docker` group;
    sshd drop-in `10-barbro.conf` (`PermitRootLogin no`, password and keyboard-interactive off, `AllowUsers ss`);
    Docker's apt repository with its key pinned inline (fingerprint `9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF
    CD88`); packages `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-compose-plugin`, `unattended-upgrades`,
    `jq`; `daemon.json` with the `local` log driver and `live-restore`; `20auto-upgrades` and the automatic reboot at
    03:30 UTC written with `defer: true`.
  - Validated locally before use: `cloud-init schema` in a `debian:trixie` container, plus an apt check that the key
    from the YAML verifies Docker's repository. The apt check caught two real defects the schema check cannot see: a
    pasted `Release.gpg` signature instead of the key, then a truncated key.
  - Server `barbro-server`: CX23, `nbg1`, Debian 13, IPv4 + IPv6, created with
    `hcloud server create … --user-data-from-file infra/cloud-init.yaml`. The first server, created in the Console,
    received empty user data (`/var/lib/cloud/instance/user-data.txt` empty) and ran with image defaults, including
    password authentication; deleted and recreated with the CLI.
  - Verified on the server: `sshd -T` (root, password, keyboard-interactive off, `AllowUsers ss`); `docker info` —
    `local true`; `unattended-upgrade --dry-run` — Debian and Debian-Security origins for `trixie`; the cloud-init log
    has only `DEBUG` lines about a metadata request retried before the network was up; root login with the right key
    rejected.
  - `infra/README.md`: runbook — create with the CLI, post-boot checks with expected output, local validation, updates.
- `barbro-docs`: `threat-model.md` (SSH 22 open as an accepted risk; deploy controls per ADR-0007; GHCR token removed
  from the secrets; rows for a replaced image and a broken release), `architecture.md` §5 (deploy, firewall, logs),
  ADR-0007 with the `adr/README.md` row. Merged via PR.
- `infra/compose.yaml`, merged via PR: `name: barbro`; shared hardening in an `x-hardening` anchor (`cap_drop: ALL`,
  `read_only`, `restart: unless-stopped`, `no-new-privileges`); images from `${API_IMAGE:?}` / `${WEB_IMAGE:?}`; ports
  `127.0.0.1:3000` (`api`) and `127.0.0.1:8081` (`web`); `stop_grace_period` 20 s for `api`, 10 s for `web`; `web`
  tmpfs `/data` with `uid=65532,gid=65532,mode=0700`. A manual run on the server returned revision `b66efb1` from
  both services with no errors in the logs.
- Smoke scripts run the images through `infra/compose.yaml` (`-p barbro-smoke`; the image of the service not under
  test is `unused`, because Compose interpolates the whole file), so CI tests the production runtime flags. Before
  this, the API smoke test ran without any hardening flags.
- Pull deploy agent (ADR-0007), merged via PR and installed on the server:
  - `infra/deploy/barbro-deploy` (bash), `barbro-deploy.service` (oneshot, user `barbro`, `SupplementaryGroups=docker`,
    `StateDirectory=barbro`), `barbro-deploy.timer` (`OnCalendar=*:0/5`).
  - Tested by Claude against mocks of GitHub, GHCR and codeload with a fake `docker` (11 scenarios: first deploy, no
    change, CI not green, health failure with rollback and `bad`, `bad` skipped, missing images, normal deploy with
    `previous`, GitHub down, pin, pin kept, pin removed); `shellcheck` clean; GHCR digest resolution and the codeload
    `infra/` extraction checked against the real services.
  - On the server: the first manual run exited silently — CI for the HEAD (merged 81 s earlier) was still running;
    fixed by logging `waiting: no successful CI run`. A pin to a SHA without images failed without changing the
    running services. The merge of the fix was deployed hands-off by the previous agent version, then the agent was
    reinstalled.

- Domain `barbro.dev`, registered 2026-10-04 with Cloudflare Registrar (zone on Cloudflare nameservers from the start);
  DNSSEC enabled, DS published in `.dev` within hours, validated (`AD: true`).
- Ingress decision: Cloudflare Tunnel instead of inbound 80/443 (ADR-0006 amendment of 2026-10-04; three options
  compared: origin certificate + range firewall, Let's Encrypt DNS-01 + range firewall, tunnel). `threat-model.md`,
  `architecture.md` (Container diagram and table: `tunnel`, `edge`, `web`), `adr/README.md` status updated.
- Edge, merged via PR:
  - `infra/edge/Caddyfile`: `admin off`, `persist_config off`, `grace_period 15s`; site `:8080`; `encode zstd gzip`;
    HSTS, CSP (`default-src 'self'`, `frame-ancestors 'none'` and others), `nosniff`, `Referrer-Policy`,
    `Permissions-Policy`, `-Server`; `/api/*` → `api:3000`, the rest → `web:8080`.
  - `edge` service in `infra/compose.yaml`: official `caddy:2.11.4-alpine` by digest, `user: 65532:65532`, `cap_add:
    NET_BIND_SERVICE`, `depends_on: api, web`, `127.0.0.1:8082`, `stop_grace_period: 20s`, tmpfs `/data`, Caddyfile
    bind-mounted read-only.
  - Measured by Claude against a real Caddy 2.11.6 with stub upstreams before use: routing, header removal, gzip; an
    `X-Forwarded-For` sent by the client is overwritten; a 502 (upstream down) carries no security headers and `Server:
    Caddy`.
  - Verified by SS locally (podman) and on the server: both revisions through the edge, headers, gzip on assets (not on
    the small `index.html`, below Caddy's 512-byte minimum), `immutable` passing through, stop in 0.5 s, no errors.
- System smoke test, merged via PR:
  - Job `system` (`needs: [image]`, builds both images from the `gha` cache with `cache-from` only), `publish` needs it;
    required check (eight in total), proven red by removing `-Server`.
  - `.github/scripts/smoke-system.sh`: the whole stack through `infra/compose.yaml`, all checks through the edge on
    `127.0.0.1:8082` — both revisions, security headers and no `Server` on the SPA and the API, gzip and `immutable` on
    an asset, no errors in edge logs after readiness. Tested by Claude on a harness (real Caddy edge and web Caddyfiles,
    stub API): green and six red cases; `shellcheck` clean. The harness found two defects in the first version: startup
    502s logged as errors by the edge, and `jq` on HTML aborting silently under `set -e`.
- Deploy agent: health check through the edge (`127.0.0.1:8082`); Compose with `COMPOSE_PROFILES=tunnel`.
- Tunnel, merged via PR:
  - Remotely managed tunnel `barbro`; public hostname `barbro.dev` → `http://edge:8080`.
  - `tunnel` service: `cloudflare/cloudflared:2026.9.3` by digest, `tunnel --grace-period 15s run --token-file
    /run/secrets/tunnel-token`, profile `tunnel`, `stop_grace_period: 20s`; top-level Compose secret `tunnel-token` from
    `/etc/barbro/tunnel-token` (root:65532, 0440).
  - The commit deployed hands-off, but by the previous agent (installed from an outdated local checkout), without the
    profile: `api`, `web`, `edge` up, `tunnel` not started; the health check did not notice (it bypasses the tunnel).
    Fixed by installing the agent from the release directory and one manual `up -d` with the profile.
  - Verified from outside: `https://barbro.dev/api/health` and `/revision` report the deployed commit over HTTP/2; all
    security headers present.
- Cloudflare zone settings: Full (strict), Always Use HTTPS, minimum TLS 1.2, TLS 1.3; HSTS only from the edge; DNS
  holds only the tunnel CNAME. HTTP → 301 and the Cloudflare certificate checked by SS.
- `infra/README.md`: Cloudflare Tunnel setup, token storage and rotation, zone settings; agent update from the release
  directory; Compose commands for the running release.
- M0 closed: the last `barbro` merge deployed hands-off with all four services (journal, containers and `/revision`
  confirmed by SS).
- M1 planned (2026-10-05, approved by SS): four increments, each a deployable PR — see Decisions.
- M1 increment 1 — persistence and logging, merged via PRs:
  - better-sqlite3 13.0.3, drizzle-orm 0.45.3, drizzle-kit 0.31. `DatabaseModule` with two providers: `SQLITE` (the raw
    connection) and `DB_CLIENT` (Drizzle over it); pragmas `foreign_keys = ON`, `journal_mode = WAL`, `busy_timeout =
    5000` run at startup; the connection is closed in `onApplicationShutdown`.
  - Schema in `apps/api/src/db/schema.ts`: `user` (integer `AUTOINCREMENT` id, `email`, `created_at`), `user_identity`
    (primary key `(provider, subject)`, cascade from `user`, index on `user_id`), `session` (`id_hash` primary key,
    cascade from `user`, `created_at` and `expires_at` set by code, index on `user_id`). Migrations `0000`–`0002` in
    `apps/api/drizzle`, applied at startup by the Drizzle migrator (folder resolved from the module file, so `src` and
    `dist` agree).
  - Tests: health 200 `{ revision, db: "ok" }` and 503 `{ revision, db: "error" }` on a closed connection (the agent's
    `curl -f` treats 503 as unhealthy, no agent change); startup fails on a missing directory and on a file that is not
    a database (proven red by removing the pragmas); migrations create the tables; cascade from `user` to
    `user_identity` and `session` (proven red with `foreign_keys = OFF`); the connection is closed on shutdown.
  - Production wiring: `DB_PATH` required, no default (a relative default put the database inside the container);
    `/data` created in the image as `65532:65532` with mode `0700`, so Docker copies owner and mode into the empty named
    volume; `api-data:/data` in `infra/compose.yaml`; `drizzle` added to `files` for `pnpm deploy`.
  - CI step `Check migrations match schema`: `drizzle-kit generate`, then `git status --porcelain apps/api/drizzle` must
    be empty — proven red with a column added without a migration. `src/db/schema.ts` excluded from coverage with a
    comment (its callbacks run only for drizzle-kit — measured).
  - Logging: nestjs-pino 5.3.1, pino 10.4, pino-http 11; `AppModule.forRoot({ logDestination })` so tests capture the
    log; allowlist serializers — `req` = id, method, path without the query string; `res` = status code. A test proves
    that a cookie, an `Authorization` header and a query-string secret never reach the log.
  - Verified on production: `/api/health` reports the merge revision with `"db":"ok"`; `barbro_api-data` mounted at
    `/data` as type `volume`; JSON logs; the Docker `local` log driver.

## Decisions

- ADR-0001 — Bar Assistant data (MIT) as the seed: all 663 recipes, status imported/reviewed, steps and descriptions
  rewritten by an LLM, no images.
- ADR-0002 — the LLM only for FR-3b: a sommelier over a deterministic list; FR-3a without the LLM.
- ADR-0003 — English data, i18n table; free text via DeepL (successor — Azure F0); no streaming; LLM behind an adapter,
  model by eval, fallback GPT-6 Luna.
- ADR-0004 — catalog tree for navigation, matching via `satisfies_parent` + a substitution table; default pantry.
- ADR-0005 — sign-in only via Google OIDC, server-side sessions in an httpOnly cookie, `user_identity` separate.
- ADR-0006 (amended 2026-09-28 and 2026-10-04) — TypeScript, Node 26 LTS, NestJS/Fastify, Vite + React SPA, SQLite +
  Drizzle, Hetzner CX23 + Docker Compose + Caddy, GitHub Actions + GHCR with pull deploy, Sentry + UptimeRobot,
  Litestream → R2, Cloudflare DNS; monorepo `barbro`; Zod via Nest's built-in Standard Schema support (no
  `nestjs-zod`); ingress through a remotely managed Cloudflare Tunnel, no inbound HTTP; domain `barbro.dev`.
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
- ADR-0007 — pull deploy: a systemd timer runs an agent every 5 minutes; target is the HEAD of `main` whose `ci.yaml`
  push run succeeded (a pin file overrides it); both `sha-<commit>` tags resolved to digests once; `infra/` of the
  same commit; health check on the revision of both services; rollback to the running revision and a `bad` mark on
  failure. Option chosen over a moving registry tag and a CI-published release manifest (which needs
  `contents: write`).
- Server:
  - Debian 13, the same distribution as the image bases; Hetzner `nbg1`; IPv4 kept (GHCR and GitHub API reachability
    over IPv6 not measured).
  - Bootstrap by `infra/cloud-init.yaml`; servers are created only with `hcloud --user-data-from-file`, never through
    the Console form.
  - Hetzner's Docker CE app image rejected: Ubuntu only, and its apt pin (priority 1 for every package from
    `download.docker.com` except `docker-ce`) freezes `containerd.io` (with `runc`), `docker-ce-cli` and the Compose
    plugin — measured: `apt upgrade` leaves them at the snapshot version.
  - Hetzner Cloud Firewall (outside the VM, unaffected by Docker's iptables rules): 22 open to the internet with
    key-only authentication — SS's IP is dynamic, accepted risk in the threat model; fail2ban not used.
  - Updates: `unattended-upgrades` for Debian and Debian-Security, automatic reboot at 03:30 UTC; Docker Engine
    upgraded manually; `live-restore` keeps containers running across daemon restarts.
  - Docker `local` log driver (rotated by size, compressed).
  - The origin IP stays out of public repos — Cloudflare is meant to hide it.
- Compose: `infra/compose.yaml` is the single runtime spec for CI smoke tests and the server; localhost ports stay
  permanently (agent health check, smoke tests); `stop_grace_period` follows the longest legitimate request (`api`:
  the 15 s LLM timeout plus a margin).
- Deploy agent: a dedicated system user `barbro` in the `docker` group — separates state and journal, not privilege
  (the group is root-equivalent); the agent does not update itself (reinstall per `infra/README.md`); unused images
  older than a week are pruned (a rollback re-pulls by digest); silent only when there is nothing to do.

- Edge (easily reversible, no ADR; one-role decision, platform engineer):
  - Official Caddy image by digest with its Caddyfile from `infra/edge/`, bind-mounted from the release directory;
    recreated on every deploy because that path changes per commit — accepted, `api` and `web` are recreated every
    deploy anyway (revision baked into the image).
  - `cap_add: NET_BIND_SERVICE` only so the kernel executes the official binary (file capability vs `cap_drop: ALL`);
    listens on 8080. The process does hold the capability (measured on podman and on the server's Docker); it only
    allows binding ports below 1024 in its own network namespace.
  - `grace_period 15s` below `stop_grace_period: 20s`; security headers and compression live in the edge, caching
    headers in `web`.
  - The real client IP (`CF-Connecting-IP` trusted only from the `tunnel` container, Fastify trusting only the edge) —
    in M1, with the rate limits.
- Tunnel (within the ADR-0006 amendment):
  - Remotely managed: one rule in the dashboard, routing stays in the edge Caddyfile; no `cert.pem` for the account on
    the Mac.
  - In the Compose profile `tunnel`, so CI never starts it; Compose ignores the missing secret file while the profile is
    inactive (measured: CI and the deploy by the previous agent).
  - Token as a file-based Compose secret, not an environment variable (visible in `docker inspect` and
    `/proc/*/environ`); rotation — recreate the tunnel.
- CI: the system smoke test is a separate job after both image legs, so a red image is not system-tested and the build
  reuses the PR's cache; `publish` waits for it.
- Deploy agent: updates are installed from the release directory of the deployed commit (the version CI checked), not
  from a local checkout; a commit whose own deploy depends on its agent change needs a one-time manual step after the
  install.
- M1 "Sign-in" (2026-10-05, approved by SS; one-role decisions, no ADR). Increments, each a deployable PR: (1) SQLite,
  Drizzle, migrations, structured logging; (2) Google OIDC sign-in — start, callback, session, `GET /api/me`, logout,
  SPA with TanStack Query, the first server secret and the rollback proof; (3) CSRF header guard on mutations and
  account deletion; (4) the real client IP behind the tunnel, a per-IP limit on the sign-in start (10/h), the Cloudflare
  rate limiting rule. Exit criterion: on `https://barbro.dev` a user signs in with Google, sees their email, signs out
  and deletes the account; tests and pipeline green; the agent's rollback proven on the server; the per-IP limit proven
  by an HTTP test and a two-IP system smoke; the Cloudflare rule active.
- Moved out of M1: the FR-3b counters (10/day per user, 200/day global, 1 per 5 s) to the FR-3b milestone — outside-in
  needs the endpoint; Litestream to M2 — accounts come back on the next sign-in, the bar is the first data a user cannot
  recreate; Tailwind and shadcn/ui to M2. Tentative roadmap, refined at each planning: M2 catalog and bar (FR-2, the Bar
  Assistant seed, Litestream), M3 "what can I make now" (FR-3a), M4 FR-3b (counters, LLM id validation, CSP, the model
  by eval). `threat-model.md` §5 updated.
- Data layer (easily reversible, no ADR):
  - ADR-0006 stands: `node:sqlite` is Stability 1.2 (release candidate) in Node 26.7, and Drizzle's `node-sqlite` driver
    exists only in `drizzle-orm@1.0.0-rc` (measured 2026-10-05). Revisit when both are stable; Drizzle isolates the
    driver.
  - Migrations run at API startup: one process on one server, no race; a failed migration fails the health check and the
    agent rolls back. A snapshot before migrations comes with Litestream in M2.
  - Integer primary keys for ids that never leave the server; an id exposed in a URL gets a separate UUIDv7 column.
    `AUTOINCREMENT` on `user`, so a hard-deleted id is never reused.
  - Session token: 256 random bits, only its SHA-256 stored (no slow hash needed for random tokens); timestamps set by
    code so tests control the clock.
  - The `better-sqlite3` build script is denied in `pnpm-workspace.yaml`: the npm tarball ships prebuilt binaries
    (`prebuilds/`, glibc and musl, x64 and arm64) loaded by `lib/binding.js`; the only build step is pnpm's implicit
    `node-gyp rebuild` for packages with `binding.gyp`, which needs Python and a compiler.
  - Health `db` means "the connection answers `select 1`", not data integrity; a database that cannot be opened fails
    the startup instead.
- Logs (easily reversible, no ADR): allowlist serializers instead of redaction — no headers, no query strings, no client
  IP, no bodies — so logs carry no personal data and size-based rotation by the Docker `local` driver (5 × 20 MB per
  container) is enough; the `threat-model.md` row on log leaks rewritten accordingly (it promised a 14-day rotation).
- Recommended for increments 2–4, decided at the step: `openid-client` for the protocol (OpenID-certified; sessions and
  tables stay ours per ADR-0005); the Google client secret as a file-based Compose secret like the tunnel token; the
  rollback proof with that secret (file missing → Compose refuses to start, a path the agent has not seen yet; empty
  file → config validation fails → health red → rollback); no Playwright E2E through Google (automated Google sign-in is
  blocked) — the callback is covered with a fake `IdentityProvider`, the adapter against a local fake IdP; the rate
  limiter (`@fastify/rate-limit` vs `@nestjs/throttler`) chosen in increment 4.

## Stack and tools

- **Runtime and tooling:** Node 26 LTS (`fnm` + `.nvmrc` locally), TypeScript 6, pnpm 12.8 (standalone install in
  `~/Library/pnpm`, kept at the `packageManager` version with `pnpm self-update`).
- **API:** NestJS 12 (ESM) on Fastify 5, Zod 4 through Nest's built-in Standard Schema validation, OpenAPI.
- **Web client:** Vite 8, React 19, TanStack Query, Tailwind, shadcn/ui, PWA; tests with Vitest, jsdom and Testing
  Library. Served by Caddy 2.11 (`barbro-web` image).
- **Data:** SQLite (better-sqlite3 13 on Node-API, prebuilt binaries from the npm tarball), Drizzle ORM 0.45 +
  drizzle-kit (migrations in `apps/api/drizzle`), Litestream (M2).
- **Logging:** pino 10 through nestjs-pino 5 (pino-http), JSON to stdout, Docker `local` log driver.
- **Infrastructure:** Docker (multi-stage, distroless and Alpine runtimes, Buildx), Docker Compose (profiles, file-based
  secrets), Caddy (edge and static); Cloudflare Registrar, DNS (DNSSEC), proxy, Tunnel (`cloudflared`), R2; GitHub
  Actions, GHCR; Sentry, UptimeRobot. Locally Docker Desktop on macOS (arm64)
  and podman on one of the machines.
- **Server:** Hetzner Cloud CX23 (`hcloud` CLI, Cloud Firewall), Debian 13, cloud-init, `unattended-upgrades`; deploy
  agent in bash + curl + jq under a systemd timer, logs in journald.
- **Quality and testing:** Biome 2.5; Vitest 4 (handles Nest decorator metadata out of the box, no SWC plugin);
  fast-check for property-based tests; Playwright; gitleaks (binary in CI); Renovate (self-hosted, own GitHub App);
  husky + lint-staged; Bruno for the API; bash + curl image smoke tests.
- **Docs repo:** markdownlint-cli2, cspell, Dependabot. Diagrams — Mermaid in Markdown.
- **Adapters:** `Translator` (DeepL → Azure), `Sommelier` (provider SDK) — DI tokens as abstract classes.

## Open questions

- Caddy 2.11.6 (on Docker Hub since 2026-10-02, cooldown over): confirm that one Renovate PR updates both
  `apps/web/Dockerfile` and the `edge` image in `infra/compose.yaml`; check the Dependency Dashboard for
  `cloudflare/cloudflared` too.
- Real client IP behind the tunnel — M1 increment 4, with the rate limits: the edge trusts `CF-Connecting-IP` only from
  the `tunnel` container (`trusted_proxies`), Fastify trusts only the edge; two clients with different IPs get different
  buckets. Keep the IP out of the logs (it is personal data).
- The deploy agent's health check does not see the tunnel. Until UptimeRobot (phase 5): consider `cloudflared --metrics`
  with its `/ready` endpoint as a cheap local check.
- Edge 502 responses (upstream down) carry no security headers and expose `Server: Caddy`: Caddy's error path skips the
  `header` directive. Empty body, low risk — decide whether a `handle_errors` block is worth it.
- CSP: strict for the empty SPA; check the browser console on `barbro.dev` for violations as the SPA grows (inline
  styles from UI libraries), loosen only per directive with a reason.
- podman: `infra/compose.yaml` does not run under podman-compose — podman rejects the tmpfs options `uid=`/`gid=` (its
  equivalent is `U`), so `CONTAINER_CLI=podman` in the smoke scripts is broken. Drop the podman claim from the scripts
  or keep a local override; podman also refuses an `amd64`-only index on arm64 without `--platform`.
- `www.barbro.dev` is not configured — decide on a redirect before phase 4.
- Cloudflare rate limiting rule on `/api/*` (threat model) — M1 increment 4; check what the Free plan allows at setup
  time.
- Tunnel token rotation (recreate the tunnel) is documented but not rehearsed.
- Rollback after a failed health check is verified only against mocks. Prove it on the server in M1 increment 2 with the
  Google client secret: file missing (Compose refuses to start — the agent has not met this path yet) and empty file
  (config validation fails, health red, rollback).
- Smoke via Compose: confirm the red check was run (`web` tmpfs without `uid`/`gid` must fail `image (web)`).
- Hetzner API token (Read & Write) on SS's Mac in `~/.config/hcloud/cli.toml` in plain text — keep or revoke between
  server rebuilds.
- Deploy failures are visible only in journald until alerting exists (`OnFailure=` or monitoring in phase 5).
- Hardening path from ADR-0007: signed GitHub artifact attestations verified on the server before deploy.
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
- `barbro-docs` `.github/dependabot.yaml`: the group pattern `cspell/dict-uk-ua` lacks the `@` and does not match
  `@cspell/dict-uk-ua`, so that dictionary would get its own PR instead of joining the `npm` group — fix the pattern.
- `barbro-docs`: `npm audit` reports 5 high — one advisory, CVE-2026-93687 (GHSA-vfj7-8cjw-p6xm, reviewed 2026-10-02):
  stack exhaustion in `braces` ≤ 3.0.3 on deeply nested brace patterns, reached via `markdownlint-cli2` → `globby` →
  `fast-glob` → `micromatch`. No patched version exists. Accepted risk: a dev-only lint tool that expands only globs
  from our own repo, so the worst case is a crashed lint run; no exposure in the deployed product (`barbro`'s lockfile
  has no `braces`). Do not run `npm audit fix --force`: its "fix" downgrades `markdownlint-cli2` to 0.0.4. Take the
  `braces` patch when it ships.
- LLM model — by the eval set (~20 EN cases, with injection); candidates GPT-6 Luna, Gemini 3.5 Flash-Lite, Claude Haiku
  4.5.
- Azure Translator F0 — test on the same phrases before the i18n milestone (needs a card).
- 18+ for an alcohol site (Legal, before phase 4). Privacy policy with the list of processors. "Source" link in the UI
  footer (AGPL section 13) — phase 4 checklist.
- Protecting cocktail names in DeepL (`tag_handling`) — verify at implementation.
- Google Gemini prices — verify on the official page before the eval.
- Organizational: Claude Code in a clone of the repo for reviews — optional.
- pnpm hung silently (no output even with `--loglevel=debug`) inside the repo when `packageManager` (12.8.2) differed
  from the installed pnpm (12.6.0), i.e. while switching versions; worked around with `pnpm self-update 12.8.2`.
  Investigate if it recurs on the next Renovate pnpm bump.
- `node:sqlite` instead of better-sqlite3 — revisit when `node:sqlite` and Drizzle's `node-sqlite` driver are stable (no
  native module in the image).
- Local dev on podman (rootless) hides permission defects that Docker shows (a root-owned bind mount, a tmpfs): a green
  local run under podman is not evidence for the server; CI on Docker is.

## Learned in this product

Level check, 2026-10-04, by SS: from the Caddy work onward (web image, smoke scripts, CI matrix, cloud-init, Compose,
deploy agent, edge, tunnel) SS ran and verified solutions that Claude built and tested, and did not master them: he
could not write those configs and scripts himself. For these items "practiced" and "measured" below mean "run and
verified", not "can do independently". SS chose to keep the pace and consolidate at the end of the product, before the
retrospective; new items are marked by the same rule.

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
  - Pace vs mastery (SS's choice, 2026-10-04): SS writes the code of his learning targets himself — Node, NestJS, React,
    the LLM features. Infrastructure glue (configs, CI, shell scripts) may come from Claude ready and tested to keep the
    pace; such items are marked "run and verified" in this section and go to the consolidation at the end of the
    product.
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
- cloud-init: runs once on the first boot; modules and stages (`users` and `write_files` before `packages`, `defer`
  for files written after packages); `cloud-init schema` checks structure only; the Hetzner metadata service
  (`169.254.169.254/hetzner/v1/userdata`) and `/var/lib/cloud/instance/user-data.txt` show what the server actually
  received — practiced.
- apt trust: a `signed-by` keyring holds a public key, not a `Release.gpg` signature; Debian 13 verifies with `sqv`,
  whose errors tell a signature packet from a truncated key; apt pinning — a priority below 100 never upgrades an
  installed package — measured.
- sshd: drop-ins in `sshd_config.d`, the first value of a keyword wins, `sshd -T` shows the effective config — practiced.
- Docker and firewalls: published ports bypass host iptables rules (`ufw`), so a cloud firewall outside the VM plus
  `127.0.0.1` bindings — understood.
- Compose: the project name comes from the directory unless `name:` is set; `${VAR:?}` is checked for the whole file,
  not only the started service; `x-` extension fields with YAML anchors; `-p` overrides `name:` — practiced.
- systemd: oneshot service plus timer, `OnCalendar` vs `OnUnitInactiveSec`, `StateDirectory`,
  `SupplementaryGroups`, journald — practiced.
- Registry API: an anonymous token plus a `HEAD` on the manifest returns `Docker-Content-Digest` without a pull;
  GitHub REST API: 60 unauthenticated requests per hour per IP (`403` when exceeded), `application/vnd.github.sha`,
  and an empty filter parameter (`head_sha=`) is ignored rather than matching nothing — measured.
- A silent no-op is undiagnosable: an agent that waits must say why — learned from the first run.
- Cloudflare Tunnel: outbound-only connector (QUIC on 7844 with a TCP fallback), remotely vs locally managed, token as
  the only secret, no inbound ports and no origin certificate; Cloudflare Registrar and DNSSEC (DS publication checked
  via RDAP and DNS-over-HTTPS) — practiced.
- Linux capabilities, refined: the kernel refuses `execve` of a binary whose file capability is outside the bounding
  set, so a high port does not help; container runtimes put `cap_add` capabilities into the permitted and effective sets
  before `exec`, so `no-new-privileges` has nothing to strip — measured on podman and Docker. A prediction from reading
  kernel source was wrong; the measurement settled it.
- Compose: profiles (inactive services are left out of `up`, `ps` and `logs`; a missing secret file of an inactive
  service is tolerated — measured), file-based secrets without Swarm (bind mount, host permissions apply), a bind mount
  from a per-release path recreates the container on every deploy — measured.
- Caddy as a reverse proxy: `reverse_proxy` overwrites an untrusted `X-Forwarded-For`; `encode` skips responses under
  512 bytes; the error path skips `header`; `grace_period` vs the stop timeout — measured.
- Smoke tests of a whole stack: startup errors from retried requests must be excluded from the log check; a parser
  (`jq`) in a command substitution under `set -e` aborts silently — learned from a test harness.
- Deploying a change to the deployer: the commit that changes the agent is deployed by the previous agent; install
  updates from the release directory, not from a local checkout — learned from the tunnel deploy.
- Claude's sandbox runs on gVisor: capability and kernel behavior measured there is not evidence for Linux.
- Node event loop: one JavaScript thread; I/O handed to the thread pool or the kernel; synchronous better-sqlite3 blocks
  the loop, but an in-process SQLite query takes microseconds — every query must be fast by construction, indexes come
  with the query — understood.
- SQLite: WAL (stored in the file), `foreign_keys` per connection, `busy_timeout`, `:memory:` for tests. better-sqlite3
  opens lazily — the constructor fails only on a missing directory, a file that is not a database fails on the first
  statement, so a statement must run at startup; a file overwritten under an open connection still answers `select 1`
  and schema queries from cache; better-sqlite3 compiles SQLite with `SQLITE_DEFAULT_FOREIGN_KEYS=1`, so a cascade test
  is proven red with `foreign_keys = OFF`, not by removing the pragma — measured on 13.0.3.
- Drizzle: schema in TypeScript, `drizzle-kit generate` writes SQL plus a `meta/` snapshot, the migrator records applied
  files in `__drizzle_migrations`; schema callbacks (`references`, extra config) run only in `getTableConfig` for
  drizzle-kit, never at runtime — measured; drift check with `git status --porcelain` (a new migration is an untracked
  file, invisible to `git diff`) — practiced.
- Data modelling: integer vs UUID by whether the id leaves the server, UUIDv7 vs v4 for index locality; `AUTOINCREMENT`
  against id reuse after a hard delete; a composite natural key `(provider, subject)`; SQLite does not index foreign-key
  columns of the child table; store the hash of a 256-bit token, plain SHA-256 suffices — understood.
- NestJS: the type next to `@Inject` is an unchecked claim — a Drizzle instance typed as a better-sqlite3 `Database`
  failed only at runtime, and a bare `catch` turned the `TypeError` into a 503; `HttpException` with an object body;
  `OnApplicationShutdown` with `enableShutdownHooks`; a dynamic root module `AppModule.forRoot(options)` for test parity
  — practiced.
- Tests: a fresh app per test when tests break it (`beforeEach`, not a shared `beforeAll`); `toContain` on an array
  compares whole elements; a log test must also assert the expected line exists, or it passes on empty output —
  practiced.
- Structured logging: JSON lines, `reqId` correlation, allowlist serializers vs redaction, `bufferLogs`, a destination
  object with `write` to capture logs in tests — practiced.
- Docker named volumes: an empty named volume takes the content, owner and mode of the image directory at the mount
  point, so `/data` is created in the image (distroless: in the build stage, then `COPY --chown --chmod`); Compose
  treats a source with `./`, `/` or `~` as a host bind mount, created root-owned — measured in CI with a temporary
  diagnostic step (image layout, plain `docker run`, `compose config` and `Mounts`).
- pnpm: pnpm itself runs `node-gyp rebuild` for a package with `binding.gyp` and no install script — read what the
  tarball ships before approving a build; `pnpm deploy` copies only `files`; a lockfile broken by a git merge is taken
  from `main` and re-resolved with `pnpm install`, never hand-merged; `packageManager` makes pnpm switch its own version
  — practiced.
- Claims about a package's install script or build flags are checked in the published tarball: Claude got both wrong
  from memory this milestone (`prebuild-install` in better-sqlite3 13, SQLite's foreign-key default) — learned.
- Not yet: OIDC flow, TanStack Query; React props, state and effects.

## Next step

1. SS: merge the `barbro-docs` PR with this `STATE.md` and `threat-model.md` (§5 milestone plan, the log leak row).
2. New chat "Phase 3, M1, increment 2: Google OIDC sign-in" (Fable 5.1 or Opus 5.5, High), one small step per message:
   short OIDC theory first (authorization code flow, `state`, `nonce`, PKCE, the `id_token` and its signature check); SS
   creates the Google OAuth client (Web application, redirect URIs for `https://barbro.dev` and local dev); then
   test-first — the `IdentityProvider` adapter, start and callback, the session cookie, `GET /api/me`, logout, the SPA
   sign-in with TanStack Query; the rollback proof with the first secret.
3. Then increment 3 (CSRF header guard, account deletion) and increment 4 (real client IP, per-IP limit on the sign-in
   start, the Cloudflare rule); M1 exit check.
4. In parallel, SS: Azure F0 test, the LLM eval set; confirm Dependabot; check the Renovate Dependency Dashboard
   (distroless, `Node.js` group, Caddy 2.11.6 in both places, `cloudflared`).
5. At the end of the product, before the retrospective: consolidation of everything marked "run and verified" — SS
   writes key pieces himself (the edge Caddyfile against `smoke-system.sh`, one smoke check, an annotated deploy agent
   cycle, the request path from the domain to the services), then the "Learned" section is re-graded.
