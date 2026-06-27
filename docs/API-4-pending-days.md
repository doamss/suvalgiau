# API 4 — `pending-days.php` (which days need a summary)

**Purpose:** lists days (per user) that have entries, so the cloud session knows which days to
summarize — without re-summarizing days that already have one.

```
GET  https://perkubulve.lt/suvalgiau/pending-days.php?key=BRIDGE_KEY
```

`BRIDGE_KEY` is the value in `suvalgiau/secret.php`. Without a valid key → **403**.

## Query parameters

| param | required | meaning |
|---|---|---|
| `key`   | yes | the bridge key |
| `all`   | no  | `0` (default) = only days **without** a summary; `1` = **all** days that have entries |
| `user`  | no  | restrict to one user id |
| `from`  | no  | `YYYY-MM-DD` lower bound (by entry date) |
| `to`    | no  | `YYYY-MM-DD` upper bound |
| `limit` | no  | max days (1–1000, default 365) |

Deleted (deactivated) entries are excluded, so an all-deleted day won't appear.

## Response

```json
{
  "count": 2,
  "submit_to": "https://perkubulve.lt/suvalgiau/submit-day-summary.php",
  "days": [
    { "user_id": 2, "user": "Domas", "date": "2026-06-27",
      "entry_count": 5, "analyzed_count": 5, "has_summary": false },
    { "user_id": 2, "user": "Domas", "date": "2026-06-26",
      "entry_count": 3, "analyzed_count": 2, "has_summary": false }
  ]
}
```

### Field meaning

- **`entry_count`** — non-deleted entries that day.
- **`analyzed_count`** — how many are analyzed/approved (status 2 or 3). Use this to decide whether
  the day is "ready" to summarize (e.g. only summarize when `analyzed_count === entry_count`).
- **`has_summary`** — always `false` with the default `all=0`; with `all=1` it tells you which
  days already have one.
- **`submit_to`** — where to POST the summary (see **API 3 — submit-day-summary.php**).

## Typical flow

1. `GET pending-days.php?key=…` → days needing a summary.
2. For each day, optionally fetch that day's entries (`pending.php` gives per-entry data, or you
   already have them), then write a few-line summary.
3. `POST submit-day-summary.php` with `{ user_id, date, description }`. That day stops appearing here.
