# ADR-0007: Pull deploy mechanism

Status: accepted
Date: 2026-10-03

## Context

ADR-0006 chose pull deploy: the server fetches images from GHCR itself, and CI has no access to the server (`threat-model.md`). On every push to `main`, after all checks, CI publishes two images — `barbro-api` and `barbro-web` — from two independent legs of the `publish` matrix. Images carry only `sha-<full commit>` tags; there is no `main` or `latest` tag. Three questions remained:

1. **Trigger.** How the server learns that a new commit on `main` is ready to deploy.
2. **Pair.** How it deploys `api` and `web` of the same commit and never one without the other — one matrix leg can fail while the other publishes.
3. **Configuration.** How `infra/` (the Compose file, later the edge Caddyfile) of the same revision reaches the server.

Constraints: one server; no GitHub or GHCR token on it — the packages are public and pulled anonymously; availability NFR 99%.

Ready-made updaters (Watchtower and similar) update each container by its own tag and do not carry configuration, so they answer none of the three questions.

## Options

1. **Moving tag in the registry.** A CI job after both legs repoints a `main` tag to `sha-<commit>` on both images. The server polls the tag, deploys when the `revision` labels of both images match, and fetches `infra/` from GitHub. Pro: the server depends mostly on GHCR; the live version is visible in GHCR. Con: a new CI job; two tags move non-atomically, so the pair needs a separate check; GitHub is still needed for `infra/`.
2. **The server asks GitHub whether the HEAD of `main` is green.** Each cycle: the SHA of `main` HEAD, then the `ci.yaml` run for that SHA (`event=push`, `branch=main`, `status=success`). If green: resolve both `sha-<commit>` tags to digests, fetch `infra/` at that commit, deploy by digest. Pro: the pair holds by construction — the run succeeds only when both legs published; no CI change; a red HEAD changes nothing on the server. Con: depends on the GitHub REST API; the unauthenticated limit of 60 requests per hour per IP bounds the polling interval.
3. **CI publishes a release manifest.** A final job creates a GitHub Release with the revision and both digests; the server polls the latest release. Pro: an explicit deploy log; rollback by picking a release. Con: CI gets `contents: write` — its first write access to the repository; a release on every merge.

## Decision

Option 2.

The deploy agent is a shell script run by a systemd timer on the host — not a container with the Docker socket mounted, which would be root-equivalent and need its own image. The agent lives in `infra/`. It runs every 5 minutes: two API calls per cycle, 24 of the 60 allowed per hour. Once a minute would need 120.

One cycle:

1. **Target.** The SHA of `main` HEAD whose `ci.yaml` push run succeeded. A pin file, when present, overrides it with a fixed SHA (manual rollback, freeze). The cycle ends if the target equals the deployed revision or is marked bad.
2. **Digests.** Resolve `ghcr.io/sergei-solonitcyn/barbro-api:sha-<target>` and `ghcr.io/sergei-solonitcyn/barbro-web:sha-<target>` to digests once; every later step uses the digests, so what was checked is what runs.
3. **Configuration.** Fetch `infra/` at the target commit from the public repository.
4. **Deploy.** `docker compose pull`, then `docker compose up -d` with image references pinned by digest.
5. **Health.** Both services must report the target revision (`/api/health`, `/revision`) within a timeout. On failure: `up -d` with the previous digests and configuration; the target is marked bad and not retried.
6. **State and logs.** The deployed and the previous revision with their digests are kept in a state file on the server; every step logs to journald.

## Consequences

- We get: hands-off deploy of every green merge to `main` within about 5 minutes; `api` and `web` always of one commit; images and configuration of one revision; automatic rollback when a new revision does not come up; no new permissions for CI and no token on the server.
- We pay: an own script, which needs its own checks (below); a dependency on the GitHub API — while it is down, deploys pause and the running version keeps working; up to 5 minutes from green CI to the server.
- Verified by three checks: the green path (a merge to `main` appears on the server with no manual action — the M0 criterion); a pin to a SHA without images (the agent stops, nothing changes on the server); a revision that fails the health check (rollback to the previous revision).
- For M1: the snapshot before migrations (`threat-model.md`) runs in the agent between `pull` and `up`.
- Accepted risk: whoever can push to GHCR as `sergei-solonitcyn` — CI's `GITHUB_TOKEN` on `main`, SS's account — can replace the image behind a `sha-<commit>` tag. The buildx provenance attached to the images is unsigned. Possible hardening, not in M0: publish signed GitHub artifact attestations in CI and verify them on the server before deploy.
