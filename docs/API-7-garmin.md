# API 7 — `garmin.php` (read Garmin wellness data)

**Purpose:** lets the cloud session read a user's Garmin daily wellness data (steps, sleep, Body
Battery, stress, HRV…) so it can combine it with the food log — e.g. when writing the daily review.

```
GET  https://perkubulve.lt/suvalgiau/garmin.php?key=BRIDGE_KEY&user=ID
```

`BRIDGE_KEY` is the value in `suvalgiau/secret.php`. Without a valid key → **403**.

## Query parameters

| param | required | meaning |
|---|---|---|
| `key`  | yes | the bridge key |
| `user` | yes | user id (Suvalgiau `users.id`) |
| `date` | no  | a single day `YYYY-MM-DD` |
| `from` / `to` | no | date range (inclusive). If neither `date` nor `from`/`to` is given → **last 30 days** |
| `raw`  | no  | `1` to also include the full Garmin payload per day (large; omitted by default) |

## Response

```json
{
  "user_id": 2,
  "from": "2026-06-08",
  "to": "2026-07-07",
  "count": 30,
  "days": [
    {
      "date": "2026-07-07",
      "steps": 8421, "distance_m": 6120, "floors": 12, "intensity_min": 45,
      "active_kcal": 620, "total_kcal_burned": 2380,
      "resting_hr": 52, "hrv_ms": 68, "stress_avg": 34,
      "body_battery_high": 88, "body_battery_low": 21,
      "sleep_score": 74, "sleep_min": 431
    }
  ]
}
```

Days are newest-first. Any metric not synced for a day is `null`. With `raw=1`, each day also gets
a `raw` object (the untouched Garmin response) — useful only if you need a field not in the columns.

## Notes
- This is **read-only**. Garmin data is written by the cron sync (`submit-garmin.php`), not the AI.
- The same per-day Garmin block is also embedded in **`day.php`** (see API-6), so when the session
  fetches a day to summarize/review it already has that day's wellness data inline — you usually
  don't need a separate `garmin.php` call unless you want a longer range at once (to read a trend).
