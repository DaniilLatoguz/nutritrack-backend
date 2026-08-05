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

Нет, не float — и хорошо, что переспросил, потому что вопрос показывает, где формулировка размылась.

Разведём три вещи:

Поля продукта (kcal_per_100g и БЖУ) — int. Обосновал ты: точность дробнее 1 ккал ложная при разбросе 5–20%.
grams — int. Кухонные весы дают целые.
Результат расчёта — не хранится нигде. Он вычисляется в момент запроса: kcal_per_100g * grams / 100. Python сам вернёт float — и это нормально, потому что значение живёт микросекунды и сразу округляется в ответ.

«Хранить точно» относилось к grams, а не к калориям. Из двух целых чисел точный результат восстанавливается всегда — поэтому хранить сами калории не нужно.

Держи текст.

markdown
## ADR-002: Numeric type for nutrition values

**Date:** 2026-08-05
**Status:** accepted

**Context:**
A product carries four numeric fields per 100 g: calories, protein, fat and
carbohydrates. Diary entries multiply these by the amount eaten and sum the
results into daily and weekly totals, so the chosen type determines both the
storage precision and the behaviour of every aggregation built on top of it.

**Options considered:**

- **Floating point (`float` / `double precision`).** The default choice and
  the fastest to compute with.
  Drawback: binary floating point cannot represent most decimal fractions
  exactly — `0.1 + 0.2 == 0.3` evaluates to `False` in Python. The error is
  negligible per value but accumulates across a summed day, and equality
  comparisons on stored values become unreliable.

- **Fixed-point decimal (`Decimal` / `numeric`).** Exact decimal arithmetic,
  the standard choice for money.
  Drawback: slower arithmetic and larger storage. The precision it buys is
  meaningless here — see the decision below.

- **Integer (`int` / `integer`).** Whole kilocalories and whole grams.
  Drawback: no sub-unit precision. Acceptable, since the source data does not
  carry that precision in the first place.

**Decision:**
Store all product nutrition values as integers: whole kilocalories and whole
grams per 100 g.

The determining argument is the accuracy of the data itself, not the capability
of the type. Nutrition values on food labels are batch averages and vary by
roughly 5–20% between batches of the same product. Storing `165.4 kcal` would
present a precision the underlying measurement does not have — it is not extra
accuracy, it is a false claim of accuracy. Nutrition guidelines are likewise
followed to the nearest tens of kilocalories per day.

Related but decided separately: the amount eaten (`grams`) is also stored as an
integer, for an unrelated reason — kitchen scales report whole grams. The two
fields share a type by coincidence, not by dependency.

Calculated values are not stored. A diary entry keeps `grams` and a reference to
the product; energy is derived on read as `kcal_per_100g * grams / 100`. The
intermediate result is a runtime float and is never persisted, which keeps
rounding out of the storage layer entirely.

**Consequences:**
- Rounding happens once, at the presentation boundary, when a daily or weekly
  total is returned. Per-entry rounding is avoided, so no drift accumulates
  across a day of entries.
- Products whose labels give fractional values (e.g. 0.5 g of fat) are rounded
  on import; the loss is well inside the natural batch variance.
- Sub-gram inputs cannot be recorded. Acceptable for food logging.
- Equality and aggregation over stored values behave exactly, with no floating
  point surprises in queries or tests.

**Revisit if:**
The project starts tracking substances measured in fractions of a gram — for
example micronutrients, vitamins or supplement dosages — where 1 g is a coarse
unit rather than a fine one. Those would need their own type decision and
should not simply inherit this one.

---

## ADR-003: Product identity — deferred

**Date:** 2026-08-05
**Status:** deferred — to be resolved in TASK-004

**Context:**
Nothing currently prevents the same product from being stored twice. Before a
technical constraint can be chosen, two things must be settled: what makes two
products the same, and how often duplicates actually occur.

**Options considered:**

- **Unique by `name` alone.** Simplest possible rule.
  Drawback: incorrect. Milk 3.2% from two manufacturers carries different
  nutrition values and must exist as two separate records. Enforcing uniqueness
  on the name would merge genuinely different products and lose data.

- **Unique by `(name, brand)`.** Matches how commercial calorie trackers model
  the domain — nutrition is tied to a manufacturer.
  Drawback: `brand` cannot be mandatory. Raw and home-made foods — an egg,
  boiled buckwheat, a home-made cutlet — have no manufacturer by nature.
  In SQL, `NULL` is not equal to `NULL`, so a `UNIQUE (name, brand)` constraint
  silently permits unlimited duplicates for every brandless product. The naive
  form of this option does not do what it appears to do.

- **Make `brand` mandatory with a placeholder value.** Would make the constraint
  work uniformly.
  Drawback: forces meaningless data into the column and makes "unknown brand"
  indistinguishable from "has no brand". A disguised `NULL`.

**Decision:**
Product identity is defined as the pair `(name, brand)`. `brand` stays optional.

No database constraint is introduced yet. Three workarounds exist for the `NULL`
problem — a `NOT NULL DEFAULT ''` column, a partial unique index over rows where
`brand IS NULL`, or `UNIQUE NULLS NOT DISTINCT` (PostgreSQL 15+) — and choosing
between them requires information the project does not have:

1. How frequently duplicates actually appear, and whether they arise from
   concurrent requests or from careless input. To be measured in TASK-004.
2. Who creates products. If only an administrator maintains the catalogue,
   duplicates are rare and correctable by hand; if every user can add products,
   the volume is different by orders of magnitude. This depends on the `User`
   model, which does not exist yet.

Deciding now would mean guessing at both.

**Consequences:**
- Duplicate products are possible until TASK-004. Accepted deliberately — the
  duplicates are the measurement.
- Text normalisation (case, spacing, `3.2%` vs `3,2%`) is a separate concern and
  is not addressed by any uniqueness constraint on raw strings.

**Revisit if / when:**
TASK-004, experiment 4 — concurrent `POST /products` requests with identical
payloads. The observed duplicate count decides which of the three constraint
forms is adopted in TASK-005.

---

## Open Questions

Decisions deliberately postponed until there is enough information.

| Question | Why postponed | When to revisit |
|---|---|---|
| Uniqueness constraint form for `(name, brand)` with optional `brand` | No data on duplicate frequency; product ownership undecided | TASK-004 (experiment 4) |
| Name normalisation before comparison (case, spacing, decimal separator) | Not solvable by a database constraint alone | When duplicates are observed |
| User-defined recipes / composite products | Requires `MealItem` and a nutrition aggregation model | When the diary is built |

---

## Adopted by Convention

Adopted as an industry standard, not driven by a measured problem in this project.

| What | Why the industry does it | No first-hand pain yet |
|---|---|---|
| | | |