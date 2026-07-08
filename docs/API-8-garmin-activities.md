# API 8 — `garmin-activities.php` (read Garmin activities)

**Purpose:** lets the cloud session read a user's Garmin activities (runs, hikes, swims, rides,
walks…) to combine with the food log and daily review.

```
GET  https://perkubulve.lt/suvalgiau/garmin-activities.php?key=BRIDGE_KEY&user=ID
```

`BRIDGE_KEY` is the value in `suvalgiau/secret.php`. Without a valid key → **403**.

## Query parameters

| param | required | meaning |
|---|---|---|
| `key`  | yes | the bridge key |
| `user` | yes | user id |
| `date` | no  | activities on a single day `YYYY-MM-DD` |
| `from` / `to` | no | date range (inclusive). If neither given → **last 30 days** |
| `type` | no  | filter to one Garmin type key, e.g. `running`, `hiking`, `lap_swimming`, `cycling` |
| `raw`  | no  | `1` to include the full activity payload per row (large; omitted by default) |

## Response

```json
{
  "user_id": 2,
  "from": "2026-06-08",
  "to": "2026-07-07",
  "count": 6,
  "activities": [
    {
      "activity_id": 1839200011,
      "activity_type": "running",
      "name": "Morning Run",
      "start_time": "2026-07-07 07:32:00",
      "activity_date": "2026-07-07",
      "duration_sec": 2740,
      "distance_m": 8120,
      "calories": 640,
      "avg_hr": 148,
      "max_hr": 171,
      "elevation_gain_m": 63,
      "avg_speed_ms": 2.96,
      "steps": 9450
    }
  ]
}
```

Newest-first. Any field not reported by Garmin for that activity is `null`.

## Notes
- **Read-only.** Activities are written by the cron sync (`submit-garmin-activities.php`).
- `activity_type` is Garmin's `typeKey` (e.g. `running`, `trail_running`, `hiking`, `walking`,
  `lap_swimming`, `open_water_swimming`, `cycling`, `strength_training`…).
- `avg_speed_ms` is metres/second — convert to pace/kmh as needed.
- The same activities are also embedded per-day in **`day.php`** (`activities` array), so a daily
  review already has that day's workouts inline without a separate call.
