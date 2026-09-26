# ADR-0001: Recipe database source

Status: accepted
Date: 2026-09-26

## Context

Section 6 of `requirements.md`: every suggested cocktail is a real recipe with verified proportions; "can be made now" is determined by exact matching without the LLM. The source therefore has to provide verified proportions, sufficient coverage for a 5–15-bottle bar, a clean license for a public service and an acceptable amount of ingredient normalization.

Coverage was measured with a script on a 12-item test bar: gin, vodka, white rum, bourbon, dry vermouth, sweet vermouth, triple sec, lemon juice, lime juice, sugar syrup, soda water, Angostura bitters. Pantry (ice, salt, sugar, water, garnish) is not counted.

## Options

1. **TheCocktailDB** (crowdsourced, 2021 dump, 574 drinks). Makeable: 19–34 depending on pantry rules, six of them recognizable. 361 ingredient names, 68 differing only by case, free-text measures ("1 1/2 oz", "2-3 oz", "4.5 cL"), garnishes in ingredient slots, 49 shots and 37 punches. The terms allow copying content obtained through the official endpoints; Premium — a one-time $10. Pro: volume. Con: quality does not meet section 6; normalizing all 361 names is on us.
2. **IBA datasets** (teijo — 77 recipes from the old list, the best structure "category + qualifier"; rasmusab — 90 recipes from 2023, measures in ml, flat names). Makeable: 13 and 4–7. The current list of 102 cocktails exists nowhere. The teijo repo has no license. Pro: professionally standardized proportions. Con: few, outdated, names to reconcile by hand.
3. **Own curated database** with IBA as the core plus home classics added by hand. Pro: full control. Con: the most expensive in time; catalog and text still from scratch.
4. **Bar Assistant data** (MIT, 663 recipes, 296 ingredients). Makeable: 20, all classics. A catalog with a hierarchy (`bourbon-whiskey` → `whiskey`), measures mostly in ml, `optional`, `substitutes` and `source` fields with a link to the primary source of every recipe (Punch 180, Imbibe 136, IBA 89, Liquor.com 22), tags on 530, ABV on 661, method on all. Pro: the highest coverage, a ready catalog model, attribution. Con: images carry third-party copyrights; step text may be close to the primary sources; tags are inconsistent.

**Difford's Guide** was considered and rejected as a data source: clause 17 of the site's terms prohibits downloading, scraping and aggregating content without written consent. Only manual verification of individual recipes is permissible.

## Decision

Option 4. Bar Assistant data as the seed of our own recipe database:

- import all 663 recipes and the ingredient catalog with its hierarchy;
- every recipe has a `status`: `imported` after import, `reviewed` after manual verification of proportions against `source`; QA is sampled, starting with the recipes that most often land in "can be made";
- step and description text is our own: one batch LLM pass rewrites `instructions` and `description` of every recipe in its own words, preserving every action and their order (rinse, muddle, dry shake are not lost); dataset text is not published; sampled review as part of `reviewed`;
- images are not imported;
- TheCocktailDB — only a pool of ideas for expansion; Difford's — only a manual reference during QA.

## Consequences

- We get: 663 recipes with attribution, a hierarchical catalog as the starting data model, the best coverage for the persona.
- We pay: tag normalization (Smokey/Smoky, Classic/Classics, 133 recipes without tags) and a decision on catalog depth — a separate ADR on the catalog model; a long ingredient list in the UI — a UX question.
- Accepted risks: MIT covers the repo but does not guarantee the cleanliness of step text, descriptions and images — mitigated by LLM rewriting of the text and dropping the images; template generation from `method` was rejected because it loses recipe-specific actions; the IBA list in the data is 89 of the current 102, the rest are added by hand.
