# NOVA_API — querying the live Nuodai UPF API (perkubulve.lt/nuodai)

A read-only **HTTP JSON API** over the **Nuodai** food dataset: ~33,750 Lithuanian grocery products
(Barbora + Rimi + the lastmile.lt aggregator of ~20 shops), each rated for ultra-processing using the
**NOVA** classification and a per-ingredient additive analysis.

You query it over HTTP — no auth, no keys, read-only. Every endpoint returns JSON (UTF-8).

## Base URL

```
https://perkubulve.lt/nuodai/api.php
```

Call it with `curl` (or your WebFetch tool):

```bash
curl -s "https://perkubulve.lt/nuodai/api.php?action=list&q=coca+cola"
curl -s "https://perkubulve.lt/nuodai/api.php?action=product&id=25821"
```

This is the live production endpoint — public, read-only, no auth.

## The flow: product → ingredients → NOVA

Just **two** calls — the product endpoint returns the NOVA verdict *and* the full ingredient
breakdown *and* the offending additives in one response:

```bash
# 1) search to find the product id
curl -s "https://perkubulve.lt/nuodai/api.php?action=list&q=lietiniai+blynai"
#    → items[].id

# 2) one call gives NOVA level (rating) + every ingredient + the NOVA-4 drivers (problematic)
curl -s "https://perkubulve.lt/nuodai/api.php?action=product&id=25821"
```

## Endpoints

### `?action=list` — search / filter / sort / paginate
Query params (all optional):

| param | meaning |
|-------|---------|
| `q` | text search. Lithuanian letters folded (`ž→z`…), split into words, **every word must appear** in the name (any order, substring). `coca-cola` → matches "Coca-Cola". |
| `cat` | category id(s), comma-separated; matches the node **and all descendants** (see `categories`). |
| `nova` | NOVA tier filter, comma-separated: `x`=unrated, `12`=NOVA 1–2, `3`=NOVA 3, `4a`/`4b`/`4c`/`4d`=NOVA 4 by severity. |
| `prob` | problematic-ingredient level: `0`=none (NOVA 1–2), `1`=some (NOVA 3), `2`=many (NOVA 4). |
| `sugar_max`, `fat_max`, `price_max` | numeric ceilings (`fat_max` = saturated fat /100; `price_max` in €). |
| `sort` | `recommended` (default), `worst`, `best`, `sugar`, `price`, `name`. |
| `page` | 1-based; **24 items per page**. |
| `ids` | explicit comma-separated id set (ignores other filters). |

Response:
```jsonc
{
  "total": 17, "page": 1, "per_page": 24, "pages": 1,
  "breakdown": { "n12": 6, "n3": 1, "n4": 9, "nx": 1 },   // NOVA split of the whole result set
  "items": [
    {
      "id": 25821, "name": "...", "brand": "...", "qty": "...", "category": "Blynai...",
      "rating": { "code": "4D", "label": "NOVA 4D", "sub": "ypač perdirbta", "band": 6 },
      "problem": { "level": 2, "label": "Daug (verčiau vengti)" },
      "n_group4": 7,
      "kcal": null, "protein": null, "fat": null, "sat_fat": null, "carbs": null, "sugar": null, "salt": null,
      "price": 17.25, "in_barbora": 0, "in_rimi": 0, "in_lastmile": 1,
      "image": "/nuodai_img/lastmile/29/29168.jpg",
      "ingredients_short": "lietiniai blynai (50 %) (karvės pienas, ..."
    }
  ]
}
```

### `?action=product&id=<id>` — full detail (NOVA + ingredients + drivers)
```jsonc
{
  "id": 25821, "name": "...", "brand": "...", "qty": "...", "pack_g": null, "unit": "g",
  "category": "Blynai, varškėčiai ir apkepai",
  "category_path": [ {"id": 5, "name": "Mėsa, žuvis ir kulinarija"}, ... ],   // breadcrumb root→leaf
  "rating":  { "code": "4D", "label": "NOVA 4D", "sub": "ypač perdirbta", "band": 6 },   // ← the NOVA verdict
  "problem": { "level": 2, "label": "Daug (verčiau vengti)" },
  "n_group4": 7,
  "nutrition": { "kcal": ..., "protein": ..., "fat": ..., "sat_fat": ..., "carbs": ..., "sugar": ..., "salt": ... },  // per 100 g/ml
  "price": 17.25,
  "ingredients_text": "lietiniai blynai (50 %) (karvės pienas, kvietiniai miltai, ...)",  // raw label
  "ingredients": [                                   // ← parsed + rated, in label order
    { "name": "karvės pienas", "e_number": null, "nova": 1, "severity": null, "flag": "p0", "is_sub": 1 },
    { "name": "Lecitinai",     "e_number": "E322", "nova": 4, "severity": 1,   "flag": "p1", "is_sub": 1 }
  ],
  "problematic": [                                   // ← only the NOVA-4 ingredients, worst-first
    { "name": "gliukozės-fruktozės sirupas", "nova": 4, "severity": 4, "class": null,
      "description": "Pigus skystas cukrus...", "harm": "Pridėtinio cukraus sirupas – ... nutukimu, 2 tipo diabetu ..." }
  ],
  "sources": [                                       // one per store carrying it
    { "source": "lastmile", "store": "Multi Cook", "title": "...", "url": "https://...",
      "price": 17.25, "image": "/nuodai_img/lastmile/29/29168.jpg" }
  ],
  "comparison": { "category": "Blynai...", "tot": 120, "items": [ /* sugar/fat/salt percentile vs category */ ] }
}
```

### `?action=categories` — the category tree
```jsonc
{ "tree": [ { "id": 5, "parent": 0, "level": 0, "name": "Mėsa...", "n": 4210 }, ... ] }
```
Flat list; build the nesting via `parent`. Only non-empty categories appear. Use an `id` as `cat` on
`?action=list` to filter to that branch.

## Reading the NOVA verdict (`rating`)

`rating.code` / `rating.label` encode the NOVA level (the raw integer isn't returned — this is richer):

| `rating.code` | NOVA | meaning | `band` |
|------|------|---------|------|
| `1-2` | 1–2 | unprocessed / minimally processed — good | 0–1 |
| `3`   | 3   | processed | 2 |
| `4A`  | 4   | ultra-processed, mild severity | 3 |
| `4B`  | 4   | ultra-processed | 4 |
| `4C`  | 4   | ultra-processed, high severity | 5 |
| `4D`  | 4   | ultra-processed, worst severity | 6 |
| `?`   | 0   | **unrated** — no readable ingredient list; class unknown (never assume "clean") | 7 |

- **"What NOVA level is this product?"** → `rating.label` (e.g. "NOVA 4D"). Anything starting `4` is ultra-processed.
- **"Why?"** → `problematic[]` — each NOVA-4 ingredient with its additive `class`, a plain `description`, and a `harm` sentence. `n_group4` is how many there are.
- **Per-ingredient flag** in `ingredients[]`: `flag` = `p0` (fine) / `p1` (concern) / `p2` (severe); `is_sub:1` = nested inside a compound ingredient.

## Notes & gotchas

- **Two steps max**: `list` (find id) → `product` (everything). No separate ingredients/nova calls needed.
- **`rating.code:"?"` = unrated**, not safe — the source had no usable ingredient list, so absence of additives is *unknown*.
- **Nutrition is often `null`** (esp. lastmile items) — the product may still be fully rated from its ingredient list.
- **`store`** on a source is the shop name for lastmile (e.g. "IKI", "Multi Cook"); empty for Barbora/Rimi.
- **Pagination**: `list` returns 24/page; read `pages`/`total` and increment `page` to walk a big result set.
- **`image`** paths are absolute (`/nuodai_img/...`); prefix with the base host for a full URL.
- Read-only and public — safe to call freely.
