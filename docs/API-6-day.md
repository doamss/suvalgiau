# API 5 — `day.php` (full detail for one day)

**Purpose:** everything the cloud session needs to write a good daily summary — every entry that
day (with analysis + photos), the day's rolled-up stats, and any existing summary.

```
GET  https://perkubulve.lt/suvalgiau/day.php?key=BRIDGE_KEY&user=ID&date=YYYY-MM-DD
```

`BRIDGE_KEY` is the value in `suvalgiau/secret.php`. Without a valid key → **403**.

## Query parameters

| param | required | meaning |
|---|---|---|
| `key`  | yes | the bridge key |
| `user` | yes | user id (from `pending-days.php`) |
| `date` | yes | `YYYY-MM-DD` |

## Response

```json
{
  "user_id": 2,
  "user": "Domas",
  "date": "2026-06-27",
  "entry_count": 5,
  "stats": {
    "total_kcal": 1940,
    "by_meal": {
      "breakfast": { "kcal": 330, "count": 1 },
      "lunch":     { "kcal": 780, "count": 1 },
      "snack":     { "kcal": 830, "count": 3 }
    },
    "avg_score": 61,
    "nova_pct": { "N1": 34, "N3": 43, "N4": 24 }
  },
  "existing_summary": null,
  "submit_to": "https://perkubulve.lt/suvalgiau/submit-day-summary.php",
  "entries": [
    {
      "id": 12,
      "eaten_at": "2026-06-27 09:40:00",
      "meal_type": "breakfast",
      "meal_label": "Pusryčiai",
      "note": "Grikių košė su uogomis",
      "analysis_status": 3,
      "analyzed": true,
      "feedback": null,
      "calories": 330,
      "AI_description": "Grikių košė su uogomis ir kava",
      "NOVA": "3",
      "score": 75,
      "place": "Sodyba",
      "photos": [ "https://perkubulve.lt/suvalgiau/uploads/2/2026-06/ab12.jpg" ]
    }
  ]
}
```

### Notes on the fields

- **`entries`** — ALL non-deleted entries that day, every status (unlike `pending.php`, which only
  returns entries still needing analysis). Ordered by time. Includes full analysis + absolute photo URLs.
- **`stats`** — server-computed, kcal-weighted, matching what the app shows:
  - `total_kcal` — sum of entry calories.
  - `by_meal` — kcal + entry count per meal type.
  - `avg_score` — average health score **weighted by calories** (only entries with both score and kcal).
  - `nova_pct` — share of calories in each NOVA band: `N1` (NOVA 1–2), `N3` (NOVA 3), `N4` (NOVA 4A–D).
- **`existing_summary`** — the current summary text for that day, or `null`. Lets you refine rather
  than start from scratch.
- **`submit_to`** — where to POST the summary (see **API 4 — submit-day-summary.php**).

## Typical flow

1. `GET pending-days.php?key=…` → which days still need a summary.
2. For each day: `GET day.php?key=…&user=…&date=…` → full detail.
3. Write a short (few-line) summary and `POST submit-day-summary.php` with `{ user_id, date, description }`.
