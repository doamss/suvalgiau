# API 6 — `day.php` (full detail for one day)

**Purpose:** everything the cloud session needs to write a good daily summary + review — every entry
that day (with analysis + photos), the day's rolled-up stats, that day's Garmin wellness and
activities, and any existing summary.

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
  "garmin": {
    "steps": 8421, "distance_m": 6120, "floors": 12, "intensity_min": 45,
    "active_kcal": 620, "total_kcal_burned": 2380, "resting_hr": 52, "hrv_ms": 68,
    "stress_avg": 34, "body_battery_high": 88, "body_battery_low": 21,
    "sleep_score": 74, "sleep_min": 431
  },
  "activities": [
    {
      "activity_type": "running", "name": "Morning Run", "start_time": "2026-06-27 07:32:00",
      "duration_sec": 2740, "distance_m": 8120, "calories": 640, "avg_hr": 148,
      "max_hr": 171, "elevation_gain_m": 63, "avg_speed_ms": 2.96
    }
  ],
  "existing_summary": { "description": null, "review": null },
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
- **`garmin`** — that day's Garmin wellness (steps, sleep, Body Battery, stress, HRV, resting HR…),
  or `null` if nothing was synced. Use it to make the `review` reflect food **and** wellness. Full
  reference: **API-7 — garmin.php** (which also serves a multi-day range for trend reading).
- **`activities`** — array of Garmin activities that started that day (runs, hikes, swims…), each with
  type, name, start time, duration, distance, calories, HR, elevation. Empty if none. Full reference:
  **API-8 — garmin-activities.php**.
- **`existing_summary`** — object `{ description, review }` (each `null` if unset). Lets you refine
  rather than start from scratch.
- **`submit_to`** — where to POST the summary + review (see **API-5 — submit-day-summary.php**).

## Typical flow

1. `GET pending-days.php?key=…` → which days still need a summary.
2. For each day: `GET day.php?key=…&user=…&date=…` → full detail (food + `garmin` + `activities`).
3. Write `description` (short food) + `review` (food ↔ wellness/activity) and `POST
   submit-day-summary.php`. For trend context across days, pull a range from `garmin.php` /
   `garmin-activities.php`.
