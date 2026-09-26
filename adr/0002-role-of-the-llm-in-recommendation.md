# ADR-0002: Role of the LLM in recommendation

Status: accepted
Date: 2026-09-26

## Context

Section 6 of `requirements.md` already excludes the LLM from ingredient matching. What remained open was what exactly the LLM does in the product. The data from ADR-0001 has flavour tags (Citrusy, Bitter, Refreshing, Smokey, Tropical, Dessert…), ABV and preparation method for almost every recipe, so a fixed set of contexts ("light", "to warm up", "for guests") reduces to filters with no semantics needed. What lacks structure is free-text mood or occasion and pairing with a dish — knowledge the data does not have.

Constraints: budget ≤ $20/month for everything, a limit of 10 LLM requests per user per day, the primary persona uses the product every evening from a phone.

## Options

1. **No LLM in MVP.** List, filter chips, "what's missing". Pro: zero inference cost, latency < 500 ms everywhere, no prompt injection. Con: the product does not train the Data / AI dimension it was chosen for, and has no food pairing.
2. **LLM for context only.** List and chips — deterministic; the "why it's worth it" explanation — from tags and description. The LLM is called only for free text or a dish: it receives the list of available cocktails and the context, picks 3–5, explains. Pro: the main scenario is free and instant; the limit and streaming concern only context requests; injection is possible through a single field. Con: some sessions will not "feel like AI".
3. **LLM on every recommendation.** Explanations always come from the model. Pro: a single tone. Con: every session costs money, the 10/day limit squeezes the main scenario; cocktail selection is no better for it.

## Decision

Option 2. The LLM is a sommelier over a deterministic list: it sees only the cocktails that can already be made, does not search, does not generate recipes and does not change proportions. It is called only when the user has provided free text or a dish. FR-3 in `requirements.md` is split into FR-3a (list, deterministic) and FR-3b (recommendation by context, LLM).

## Consequences

- We get: two API paths with different NFRs; LLM cost is counted per context request, not per session; a single entry point for prompt injection; a clear contract for the prompt — the input is limited to the candidate list.
- We pay: tag normalization for filter chips (shared work with ADR-0001); maintaining two explanation mechanisms.
- Accepted risk: users may not perceive the product as AI if they rarely enter context. Verified by analytics after release; if confirmed, option 3 can be enabled without an architecture change.
- Input for the LLM provider choice: one request ≈ system prompt + candidate list (20–100 cocktails: name, tags, short description; roughly 1–5k tokens) + context + a 300–500-token response; a ceiling of 200 requests/day at 20 active users.
