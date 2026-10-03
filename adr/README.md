# Architecture decision records

One file per decision; an ADR is written only for decisions that are expensive to change. Format: context, options with trade-offs, decision, consequences (what we get, what we pay, accepted risks).

| ADR                                                           | Decision                                                                                                           | Status   | Date       |
|---------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|----------|------------|
| [0001](0001-recipe-database-source.md)                        | Bar Assistant data (MIT) as the seed of the recipe database; step text rewritten, no images                        | accepted | 2026-09-26 |
| [0002](0002-role-of-the-llm-in-recommendation.md)             | The LLM only for free-text context (FR-3b), as a sommelier over a deterministic list                               | accepted | 2026-09-26 |
| [0003](0003-data-language-translation-and-llm-integration.md) | English canonical data with an i18n table; DeepL for free text; no streaming; LLM behind an adapter, model by eval | accepted | 2026-09-26 |
| [0004](0004-ingredient-catalog-model-and-matching-rules.md)   | Catalog tree for navigation; matching via `satisfies_parent` edges and a substitution table; default pantry        | accepted | 2026-09-26 |
| [0005](0005-authentication.md)                                | Sign-in via Google OIDC only; server-side sessions in an httpOnly cookie; identity separated from user             | accepted | 2026-09-26 |
| [0006](0006-stack-and-hosting.md)                             | TypeScript, NestJS on Fastify, Vite + React SPA, SQLite + Drizzle, Hetzner CX23 with Docker Compose, pull deploy   | accepted | 2026-09-26 |
| [0007](0007-pull-deploy-mechanism.md)                         | Pull deploy: a systemd agent deploys the green HEAD of `main` by digest; health check with automatic rollback      | accepted | 2026-10-03 |
