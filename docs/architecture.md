# Architecture Decision Log — NutriTrack

Records of significant architectural decisions made in this project.
Each decision is documented as an ADR (Architecture Decision Record).

---

## ADR-001: Store nutrition values per 100 g

**Date:** 2026-08-05
**Status:** accepted

**Context:**
A product's nutrition values must be stored in a normalized form so that
entries for different products can be summed into a daily total. Users eat
arbitrary amounts, so the stored values cannot be tied to whatever quantity
happened to be measured — every entry has to be reducible to one common base.

**Options considered:**

- **Per 100 g.** A single base unit for every product; a diary entry is
  `value * grams / 100`. Also the labelling standard used on food packaging
  worldwide, so imported data needs no conversion.
  Drawback: the user must know the weight of what they ate — "one egg" or
  "a can of cola" cannot be entered directly.

- **Per serving.** Store values for a named portion ("can", "slice", "bottle").
  Closer to how users think about food.
  Drawback: a serving has no fixed size — a bottle can be 500 ml or 2 l — so
  two products' servings are not comparable and cannot be summed without an
  extra conversion step. Partial servings ("one and a half") reintroduce the
  same problem.

- **Both.** Per-100 g base plus a list of named servings per product
  (the MyFitnessPal model).
  Drawback: requires an additional entity and its own CRUD, admin UI and
  validation before a single diary entry can be recorded. Too much surface
  for the first working version.

**Decision:**
Store all nutrition values per 100 g: `kcal_per_100g`, `protein_per_100g`,
`fat_per_100g`, `carbs_per_100g`. Diary input is accepted **in grams only**.
Volume input (ml) is out of scope — 1 ml equals 1 g only for water, and
treating them as equal would introduce a silent error of up to 8% on common
liquids (milk ~1.03 g/ml, oil ~0.92 g/ml). No unit conversion is better than
a wrong one.

**Consequences:**
- The user must know the weight of what they ate; countable items such as
  eggs or packaged drinks cannot be logged without weighing them first.
- Liquids are entered by weight or not at all.
- In exchange, every diary entry is exact, and any two products can be summed
  without conversion.
- Nutrition data imported from packaging labels maps directly onto the model.

**Revisit if:**
A data source appears that supplies package weight together with the product —
for example barcode scanning, where the scanned item already carries a known
net weight. At that point named servings can be added on top of the per-100 g
base without changing it.

---

## ADR-002: <decision title>

**Date:**
**Status:** accepted
**Context:**
**Options considered:**
**Decision:**
**Consequences:**
**Revisit if:**

---

## ADR-003: <decision title>

**Date:**
**Status:** accepted
**Context:**
**Options considered:**
**Decision:**
**Consequences:**
**Revisit if:**

---

## Open Questions

Decisions deliberately postponed until there is enough information.

| Question | Why postponed | When to revisit |
|---|---|---|
| | | |

---

## Adopted by Convention

Adopted as an industry standard, not driven by a measured problem in this project.

| What | Why the industry does it | No first-hand pain yet |
|---|---|---|
| | | |