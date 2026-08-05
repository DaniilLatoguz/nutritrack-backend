# Domain Model — Scope: Product

Scope of this document is limited to the `Product` entity and its endpoints.
`User`, `Meal` and `MealItem` are intentionally out of scope — see Open
Questions in `architecture.md`.

## Entity: Product

A reference record describing the nutritional value of a food item.
All values are stored per 100 g (ADR-001) as integers (ADR-002).

| Field | Type | Nullable | Constraints | Notes |
|---|---|---|---|---|
| `id` | int | no | PK, auto-increment | Server-generated, never accepted from the client |
| `name` | str | no | 2–200 chars | Includes variety where relevant ("Apple Golden") |
| `brand` | str | yes | max 100 chars | Absent for raw and home-made foods (ADR-003) |
| `kcal_per_100g` | int | no | 0–900 | Upper bound: pure fat is ~900 kcal/100 g |
| `protein_per_100g` | int | no | 0–100 | Grams per 100 g |
| `fat_per_100g` | int | no | 0–100 | Grams per 100 g |
| `carbs_per_100g` | int | no | 0–100 | Grams per 100 g |

**Model-level rule:** `protein + fat + carbs <= 100`, since the macronutrients
of 100 g of product cannot exceed 100 g.

## Endpoints

| Method | Path | Success | Errors |
|---|---|---|---|
| POST | `/products` | 201 | 422 |
| GET | `/products?search=&limit=&offset=` | 200 | 422 |
| GET | `/products/{product_id}` | 200 | 404 |
| PATCH | `/products/{product_id}` | 200 | 404, 422 |
| DELETE | `/products/{product_id}` | 204 | 404 |

`limit` defaults to 20 and is capped at 100.
`search` performs a case-insensitive substring match on `name`.

## Notes

- No uniqueness constraint is enforced at this stage (ADR-003). Duplicate
  products are possible by design until TASK-004 measures how often they occur.
  A `409 Conflict` response will be introduced together with the constraint.
- Serving sizes and volume input (ml) are out of scope (ADR-001).
- `kcal_per_100g` is stored rather than derived from macronutrients. Label
  values do not reliably match the 4/4/9 kcal-per-gram calculation, so the
  declared figure is treated as source data.