# ADR-0005: Authentication

Status: accepted
Date: 2026-09-26

## Context

FR-1: sign-up, sign-in, account deletion with all data. Personal data — only email and bar contents. The primary persona signs in from a phone in the evening. Budget ≤ $20/month, one developer.

What we protect: not the account (nobody wants a list of bottles) but the LLM and DeepL quota — the 10-requests-a-day limit per user works only when a fake account has a price. Second requirement: do not store what can leak — passwords.

At the product's scale managed services are free (Clerk 50k MRU, Supabase Auth 50k MAU, Auth0 25k MAU), so cost is not a criterion.

## Options

1. **OAuth via Google, own sessions.** We write the authorization code flow, a cookie session, user tables. We do not write passwords, resets, email sending. Pro: the price of a fake account is a Google account; one-tap sign-in on a phone; zero user secrets on our side; does not constrain the stack. Con: dependency on Google; a user without a Google account — practically a non-case for the persona.
2. **Passwordless email (magic link / code), own implementation.** Pro: no third-party identities. Con: an email provider, SPF/DKIM/DMARC, spam — an operational burden; the worst UX on a phone; a fake account costs one disposable mailbox — needs rate limiting and captcha.
3. **Managed service (Supabase Auth, Clerk, WorkOS AuthKit, Auth0).** Pro: OAuth, magic link, passkeys and UI in an hour. Con: the vendor as a processor of personal data (DPA); Supabase Auth drags Supabase Postgres along — that is a stack decision; Clerk pushes toward Next.js and has a painful exit; Auth0 is overkill.
4. **Email + password, hand-rolled.** The largest attack surface with no advantages over option 2. Rejected.

## Decision

Option 1: sign-in only via Google (OpenID Connect, authorization code flow) for MVP. Sessions are server-side, in an httpOnly cookie. We store the email, Google's `sub` and the creation date; no passwords.

The data model separates identity and user from day one: `user(id, email, created_at)`, `user_identity(user_id, provider, subject)`. Apple Sign In, passkeys or magic link are added as a new `provider` row, without a migration.

Account deletion (FR-1) is hard: `user`, `user_identity`, `user_bar_item`, limit counters; no soft delete.

Apple Sign In is not mandatory on the web (Apple's requirement applies to iOS apps in the App Store) and needs the Apple Developer Program — out of MVP.

## Consequences

- We get: the smallest attack surface, zero email infrastructure in operation, the most expensive fake account among the available options, the fastest sign-in from a phone.
- We pay: dependency on Google; Google as a third party in the privacy policy; OAuth consent screen setup in Google Cloud (for the email/profile scopes verification is not required — check during implementation).
- Accepted risks: a user without a Google account cannot sign up — accepted for MVP, removed by a second provider; the app being banned by Google — unlikely for the email/profile scopes, recovered by adding another provider.
- Threat model (phase 2, later): rate limiting on account creation and on FR-3b on top of the per-user limit; CSRF protection of the OAuth flow (`state`); session lifetime.
