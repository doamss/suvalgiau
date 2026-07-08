# API 5 — `submit-day-summary.php` (write a daily summary + review)

**Purpose:** the cloud Claude Code session posts a short food summary (`description`) and a longer
food↔wellness/activity review (`review`) for a given user + day. The app shows them when you tap that
day's header. Re-posting the same (user_id, date) overwrites; the two fields update independently.

```
POST https://perkubulve.lt/suvalgiau/submit-day-summary.php?key=BRIDGE_KEY
Content-Type: application/json
```

`BRIDGE_KEY` is the value in `suvalgiau/secret.php` (same key as the other endpoints). Without a
valid key → **403**.

## Request body

One object, or a batch under `items`:

```json
{
  "items": [
    {
      "user_id": 2,
      "date": "2026-06-27",
      "description": "Subalansuota diena: pusryčiai ir pietūs daugiausia NOVA 1–2, vienas ultra-perdirbtas užkandis vakare. Bendra dienos kokybė gera.",
      "review": "~1940 kcal, kokybė gera. Miego balas 74, Body Battery iki 88, HRV 68, pulsas ramybėje 52 — tvarkinga naktis po švaresnės dienos. 8400 žingsnių, jokios treniruotės."
    }
  ]
}
```

### Fields (per item)

| field | type | notes |
|---|---|---|
| `user_id`     | int  | **required** — the `user_id` you got from `pending.php` |
| `date`        | text | **required** — `YYYY-MM-DD` (the day being summarized; alias: `summary_date`) |
| `description` | text | a **short** food summary — a few lines. Trimmed to 2000 chars. (alias: `summary`) |
| `review`      | text | a **longer daily review** combining food + Garmin wellness/activity. Trimmed to 6000 chars. |

Keep `description` to ~2–3 sentences (the UI shows it in a small panel). `review` can be longer —
it's shown under the description. Send both in one call, or just one; **omitting a field keeps the
previously stored value** (they update independently, not wiped). Pull the day's food + Garmin data
with `day.php` (API-6) first, then write both.

## Response

```json
{ "ok": true, "saved": [ { "user_id": 2, "date": "2026-06-27" } ], "skipped": [] }
```

- **`saved`** — the (user_id, date) pairs written (inserted or updated).
- **`skipped`** — items missing a valid `user_id` or `date`.

## Notes
- Upsert keyed on (user_id, date) — one summary per day; re-posting replaces it.
- This is independent of per-entry analysis (`submit-analysis.php`); you can summarize a day
  whenever you like, e.g. after analyzing all of that day's entries.

## Minimal example

```bash
curl -s -X POST "https://perkubulve.lt/suvalgiau/submit-day-summary.php?key=BRIDGE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id":2,"date":"2026-06-27","description":"Gera diena: daug daržovių, mažai cukraus.","review":"Miego balas 90, HRV 47, pulsas ramybėje 55 — kyla po švaresnių dienų."}'
```
