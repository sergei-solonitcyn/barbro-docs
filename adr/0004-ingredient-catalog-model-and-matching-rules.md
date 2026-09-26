# ADR-0004: Ingredient catalog model and matching rules

Status: accepted
Date: 2026-09-26

## Context

Section 6 of `requirements.md`: "can be made now" is determined by exact matching of the bar's ingredients against the recipe, without the LLM. FR-2: for base spirits a category is enough. The catalog from ADR-0001 (Bar Assistant, 296 ingredients) has a tree via `parent_id` up to 4 levels deep; recipes reference generic nodes in 686 of 2,984 positions (`gin` 135, `simple-syrup` 139, `sweet-vermouth` 70, `mezcal` 63, `bourbon-whiskey` 50).

The tree is a taxonomy, not interchangeability: the children of `simple-syrup` are orgeat, honey, cinnamon syrup; the child of `mezcal` is tequila; the children of `vodka` are vanilla, citron, sage-infused. The rule "a child covers its parent" is correct for whiskey, brandy, sparkling wine, curaçao and wrong for the four most frequent nodes. Pantry is not distinguished in the data (`sugar` 14, `water` 8, `salt` 5 are ordinary ingredients); garnish is a separate text field; `optional` on 31 positions, `substitutes` on 55 (at recipe level).

## Options

1. **Flat catalog, exact match by id.** Always correct, zero manual work. Con: a user with bourbon does not see the Manhattan; 296 items with no grouping in the UI.
2. **Tree as is, child covers parent.** Zero work, uses the existing structure. Con: false matches on high-frequency nodes — a direct violation of section 6. Rejected.
3. **Tree for navigation, explicit rules for matching.** Exact match; generalization only along edges marked `satisfies_parent`; a separate substitution table with direction and quality. Con: auditing ~25 parent nodes and the substitution table by hand.

## Decision

Option 3.

**Entities** (conceptual, not tied to a DBMS):

- `ingredient(id, name, parent_id, category, abv, is_pantry, satisfies_parent)`
- `substitution(from_id, to_id, quality)` — `quality ∈ {exact, approximate}`, directed: "I have `from`, the recipe wants `to`"
- `recipe(id, name, method, glass, abv, status, source_url, …)`, `recipe_ingredient(recipe_id, ingredient_id, amount, unit, optional, note)`
- `tag(id)`, `recipe_tag` — a controlled vocabulary; mapping at import (Smokey/Smoky → smoky, Classic/Classics → classic, Herbal/Herbacious → herbal)
- `translation(entity, entity_id, field, locale, text)` — from ADR-0003
- `user_bar_item(user_id, ingredient_id, status)` — `status ∈ {available, out}`

**Matching rule.** A recipe position `r` (not `optional`, not pantry) is covered if the bar has an available ingredient `b` such that one of the following holds:

1. `b = r`;
2. `r` is an ancestor of `b` and every edge on the path from `b` to `r` has `satisfies_parent = true`;
3. a `substitution(b → r)` exists.

A recipe "can be made" if all its positions are covered; if at least one is covered via `approximate`, it is marked "≈" in the list. The same function with a counter of uncovered positions gives S4 "what's missing".

**Pantry.** `is_pantry = true` for ice, water, sugar in all its forms (sugar, sugar cube, superfine, powdered), salt, pepper. Present by default; the user can disable it: no `user_bar_item` row for a pantry ingredient means "available", a row with `status = out` means "disabled". Egg white, mint, cream, fresh fruit are ordinary ingredients.

**Audit at import** (manual work by SS, one pass): for every parent node referenced by recipes, set `satisfies_parent` on its children; re-parent `tequila` from under `mezcal` to a common parent (agave spirits); flavoured and infused variants (`coconut-rum`, `vanilla-vodka`, `bacon-fat-infused-bourbon`) do not cover the base node. Starting substitution table: the 55 recipe-level `substitutes` from the data plus classic pairs (rye ≈ bourbon, cointreau = triple sec, dry curaçao = triple sec).

## Consequences

- We get: deterministic matching that does not lie; one function for S1 and S4; a catalog with grouping for the UI; substitution data that can be shown to the user.
- We pay: the edge audit and the substitution table are manual work; two sources of truth for "covers" (edges and substitutions) — needs a test that `reviewed` recipes give the expected result on the reference bar.
- Accepted risks: the audit is subjective (does Punt e Mes cover sweet vermouth?) — recorded conservatively, extended on complaints; a long list in the UI — a UX question, solved by grouping along the tree and search.
- The coverage from ADR-0001 (20 recipes on the reference bar) was counted with option 2's rule; there were no false matches on that bar, but the figure must be recomputed after the audit.
