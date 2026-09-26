# BarBro: Requirements

Updated: 2026-09-26
Status: approved; FR-3, NFRs and open questions refined per ADR-0001 … ADR-0006 and `threat-model.md`

## 1. Problem

Someone who enjoys making cocktails at home has the bottles but not the knowledge to put them to use. Hypotheses (validated with real users after release):

- **H1.** The user does not know which cocktail is worth making from what they have, and why that one.
- **H2.** The user does not understand what goes with what: a cocktail for a meal, a mood, an occasion.

Consequence: bottles gather dust, the repertoire is two or three familiar cocktails, and new attempts end in a poor result with no understanding of why.

## 2. Users

A public web service; each user keeps their own bar.

**Primary persona:** a cocktail novice with a small home bar (5–15 bottles, basic ingredients) who wants to make something new and tasty without studying bartending literature. Uses the product in the evening, from a phone or a laptop.

## 3. Scenarios

In priority order:

| #  | Scenario               | Description                                                                                 |
|----|------------------------|---------------------------------------------------------------------------------------------|
| S1 | What to make now       | The user sees which cocktails can be made from what they have, and why each is worth trying |
| S2 | For a mood or occasion | The same, with context: "something light for the evening", "for guests", "to warm up"       |
| S3 | With food              | The same, with a dish as context: "with steak", "with a cheese board"                       |
| S4 | What's missing         | Which single bottle to buy to unlock the most new cocktails                                 |
| S5 | Teach me               | Why ingredients combine; principles of cocktail balance                                     |

S2 and S3 are not separate features but **context** for the S1 recommendation. The fixed set of S2 contexts is implemented as filters (FR-3a); free-text S2 and S3 go through the LLM (FR-3b).

## 4. MVP scope

**In MVP:** account, bar, list of available cocktails with filters (FR-3a), recommendation by context (FR-3b).

**Out of MVP, in order of appearance:**

1. S4 "What's missing" — first milestone after release.
2. S5 "Teach me".
3. A paid tier with higher recommendation limits — a monetization hypothesis, not before demand appears.
4. Volumes, opening dates, bottle photos, history of cocktails made.

## 5. Functional requirements

### FR-1. Account

- Sign-up and sign-in; the authentication method is chosen in architecture.
- Deletion of the account together with all its data at the user's request.

### FR-2. Bar

- Add an ingredient from the catalog (spirits, liqueurs, syrups, juices, garnishes), remove it, mark it as "out".
- Ingredients come from a shared catalog rather than free text, so that matching against recipes is deterministic.
- For base spirits a category ("gin", "bourbon") is enough; the brand is optional.

### FR-3a. List of available cocktails

- Cocktails that can be made from what is available now, with the recipe: ingredients with proportions, steps.
- Optional filter by a fixed set of contexts (tentatively: light, strong, refreshing, warming, for guests; the exact set is defined in UX) based on tags, ABV and preparation method.
- A short "why it's worth trying" — derived from the recipe's tags and description.
- Computed deterministically, without the LLM, without a limit.

### FR-3b. Recommendation by context

- The user describes a mood, occasion or dish in free text.
- The LLM receives only the FR-3a list and the context, picks 3–5 cocktails and explains why they suit the bar and the context.
- The LLM does not see recipes outside the list, does not generate recipes and does not change proportions.
- The context is translated into English, the LLM works in English, the explanation is translated into Ukrainian (ADR-0003). Cocktail names are not translated.
- The response is returned in full, with a waiting indicator; streaming is not used.
- Limit of 10 requests per user per day; the remainder is shown to the user.

## 6. Key requirement: recipes are not invented

Every suggested cocktail is a real recipe with verified proportions from the product's recipe database (source — ADR-0001). "Can be made now" is determined by exact matching of the bar's ingredients against the recipe, without the LLM. The LLM picks from the already-makeable cocktails for the context and explains the choice (ADR-0002); the LLM does not generate recipes and does not change proportions.

Rationale: the primary persona is a novice who cannot tell wrong proportions from right ones; invented recipes would destroy trust in the product after the first failed cocktail.

## 7. Non-functional requirements

| NFR                              | Value                                                                                             | Rationale                                                                                    |
|----------------------------------|---------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| Users                            | 100 registered in the first 3 months, up to 20 active per day                                     | No marketing; overestimating would justify unnecessary complexity                            |
| Peak load                        | up to 5 requests/s                                                                                | Friday evening, a few people at once                                                         |
| Latency, regular pages and FR-3a | p95 < 500 ms                                                                                      | Simple DB queries                                                                            |
| Latency, FR-3b                   | p95 < 10 s to the full response                                                                   | Translation + LLM + translation; for a 10-requests-a-day feature a waiting indicator is fine |
| Availability                     | 99% monthly (≈ 7 h of downtime)                                                                   | One server without failover, deploys without maintenance windows                             |
| Budget                           | ≤ $20/month: hosting, domain, LLM                                                                 | The LLM is the main variable; the request limit protects the budget                          |
| LLM cost                         | ≤ 10 context requests (FR-3b) per user per day; ceiling 200 requests/day                          | Budget protection against abuse in a free public service; FR-3a is free                      |
| Backup                           | RPO 24 h, RTO 4 h                                                                                 | Losing a day of inventory is annoying, not catastrophic                                      |
| Personal data                    | Only email and bar contents; privacy policy; account deletion                                     | A public service with email — GDPR regardless of scale                                       |
| Clients                          | Mobile and desktop browsers, current versions                                                     | The primary persona uses a phone in the evening                                              |
| Languages                        | Ukrainian UI; English data with an i18n layer; nomenclature translated once, free text at runtime | ADR-0003; other languages can be added without a rebuild                                     |

## 8. Open questions (for architecture and implementation)

- Concrete LLM model — by eval results (candidates and costs in ADR-0003).
- Response cache for FR-3b — after data on repeated contexts.

Resolved: recipe database source and license (ADR-0001), role of the LLM (ADR-0002), data language, translation and LLM integration (ADR-0003), ingredient catalog model and matching rules (ADR-0004), authentication (ADR-0005), stack and hosting (ADR-0006); protection of the free-text context field — `threat-model.md`.
