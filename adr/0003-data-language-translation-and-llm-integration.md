# ADR-0003: Data language, machine translation and LLM integration

Status: accepted
Date: 2026-09-26

## Context

ADR-0002 limited the LLM to the FR-3b scenario: selection from a deterministic list and an explanation for a free-text context. Left to decide: which language to store data in, how to serve the Ukrainian UI, how to integrate the LLM and which model to take within the ≤ $20/month budget (≈ $10 of it for external services).

The data source (ADR-0001) is in English; LLMs are strongest in English. One FR-3b request ≈ 2,500 input tokens and ~500 output; ceiling 200 requests/day (6,000/month), realistically ~1,000/month.

## Options

### Data language

1. Canonical data in Ukrainian — translate the whole dataset, the LLM works with a Ukrainian list. Con: every source update has to be translated again; worse LLM quality.
2. Canonical data in English, an i18n layer for the UI. Pro: source and LLM in their native language; other languages can be added without a rebuild.

### Translating free text (user context → LLM → explanation to the user)

1. The LLM reads and writes Ukrainian itself. Pro: one call, no translator. Con: the quality of Ukrainian in cheap models is unknown; Ukrainian output is more expensive in tokens.
2. LibreTranslate (Argos), self-hosted. Tested on domain texts: unusable in both directions (an Old Fashioned recipe came back as nonsense about "a snowstorm, a sugar cube and a pair of mountain vests"; "сирна дошка" / cheese board → "blue board").
3. DeepL API Developer: 1,000,000 characters once, no monthly reset; then the Growth plan with a fixed monthly fee. Tested: uk→en flawless, en→uk usable (shake → "змішайте" / "mix", terms drift, cocktail names get transliterated).
4. Azure AI Translator F0: 2 million characters/month free, then $10/million; requires a card. Not tested.
5. Google Cloud Translation: 500 thousand characters/month free, then paid with no automatic cut-off. Not tested.

One FR-3b request through the translator ≈ 2,100 characters (context ~150, response ~2,000). At the realistic volume that is 2.1 million characters/month, at the ceiling — 12.6 million.

### Streaming the LLM response

1. Streaming (as in requirements v1). Con: SSE on the backend, a partial response on the frontend, incompatible with structured output — JSON with cocktail ids cannot be shown token by token.
2. A full response with a loading indicator. For a 10-requests-a-day feature, 5–8 s of waiting is acceptable.

### LLM integration

1. The provider's SDK directly in product code.
2. A thin in-house adapter with the interface "candidates + context → chosen ids + explanations"; provider and model — configuration.
3. A gateway (OpenRouter, Vercel AI SDK etc.) — one more layer and one more account for a switch that MVP does not need.

### Model

Prices as of 2026-09-26; Anthropic and OpenAI — official pages, Google — secondary sources, verify before implementation.

| Model                                               | $/M in / out | $/request | Ceiling 6,000/month | Realistic 1,000/month |
|-----------------------------------------------------|--------------|-----------|---------------------|-----------------------|
| GPT-6 Luna                                          | 0.10 / 0.50  | 0.0005    | $3                  | $0.5                  |
| Gemini 2.5 Flash-Lite                               | 0.10 / 0.40  | 0.00045   | $2.7                | $0.45                 |
| Gemini 3.5 Flash-Lite                               | 0.30 / 2.50  | 0.002     | $12                 | $2                    |
| Gemini 3.8 Flash (introductory, ×2 from 2027-01-01) | 0.75 / 3.75  | 0.0038    | $22                 | $3.8                  |
| Claude Haiku 4.5                                    | 1 / 5        | 0.005     | $30                 | $5                    |
| Claude Sonnet 5, GPT-6 Sol                          | 2 / 10       | 0.01      | $60                 | $10                   |

At the ceiling only Luna and Flash-Lite fit the budget. In reasoning models "thinking" tokens are billed as output — reasoning is disabled or capped.

## Decision

1. **Canonical data language — English.** Translations live in a separate table `(entity, field, locale, text)`, not in `*_uk` columns. MVP — `uk` only.
2. **Nomenclature is translated once** when the database is populated: ingredients (296), tags, descriptions, step templates — the LLM in batch mode with a glossary of bar terms (shake / stir / dry shake / twist / simple syrup / club soda etc.), then manual review by SS. Cocktail names are not translated, neither in data nor in the UI.
3. **Free text is translated by DeepL API Developer** in both directions, behind a `Translator` adapter with the interface `translate(text, from, to)`; the whole million characters goes to runtime (nomenclature is translated by the LLM, item 2). **Successor after exhaustion — Azure AI Translator F0** (2 million characters/month, resets monthly): tested on the same domain phrases before phase 3 starts, switched via adapter configuration. The LLM works in English only. Cocktail names in the LLM response are protected from translation with DeepL markup (verify during implementation) or by substitution after translation.
4. **No streaming.** FR-3b returns a full structured response; NFR — p95 < 10 s to completion.
5. **The LLM behind a thin `Sommelier` adapter** with the interface `recommend(candidates, context) → {picks: [{id, reason}]}`: structured output (JSON), reasoning off, low temperature, the 10/day per-user limit enforced on the backend, provider keys in secrets.
6. **The model is chosen by an eval set** of ~20 cases in English (bar × context, including prompt-injection attempts in the context field) among GPT-6 Luna, Gemini 3.5 Flash-Lite, Claude Haiku 4.5. The chosen model is recorded in `STATE.md`, not in an ADR. Without an eval the starting model is GPT-6 Luna (×3 budget margin at the ceiling).

## Consequences

- We get: a cheap and clear FR-3b chain — DeepL uk→en → LLM (EN) → DeepL en→uk; switching the model or the translator is a config and adapter change; an i18n model ready for other languages.
- We pay: two external keys and two points of failure in FR-3b; the glossary and the nomenclature review are manual work for SS; loss of nuance in double-translated explanations.
- Accepted risks:
  - DeepL Developer is a one-off million characters: ~475 full FR-3b requests, i.e. weeks of public life, not months. The counter from `/v2/usage` goes into monitoring with an alert at 80%; the global daily FR-3b limit (threat model) is tied to the translator's remaining quota. Azure F0 — ~950 requests/month free, requires a card; above that — $10/million characters, and translation, not the LLM, becomes the main budget variable.
  - DeepL's en→uk flattens shake/stir and does not hold terms — acceptable for explanations, unacceptable for recipe steps; steps are translated once with review, not at runtime.
  - Selection quality of cheap models — verified by the eval before the choice is fixed.
  - Google prices are not verified on the official page.

Supersedes in `requirements.md` v1: the streaming requirement and "first token < 2 s".
