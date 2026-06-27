# API 3 — `submit-day-summary.php` (write a daily summary)

**Purpose:** the cloud Claude Code session posts one short summary for a given user + day. The app
shows it when you tap that day's header. Re-posting the same (user_id, date) overwrites it.

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
      "description": "Subalansuota diena: pusryčiai ir pietūs daugiausia NOVA 1–2, vienas ultra-perdirbtas užkandis vakare. Bendra dienos kokybė gera."
    }
  ]
}
```

### Fields (per item)

| field | type | notes |
|---|---|---|
| `user_id`     | int  | **required** — the `user_id` you got from `pending.php` |
| `date`        | text | **required** — `YYYY-MM-DD` (the day being summarized; alias: `summary_date`) |
| `description` | text | a **short** summary — a few lines. Trimmed to 2000 chars. (alias: `summary`) |

Keep `description` to ~2–3 sentences — the UI shows it in a small panel sized to the text.

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
  -d '{"user_id":2,"date":"2026-06-27","description":"Gera diena: daug daržovių, mažai cukraus."}'
```
