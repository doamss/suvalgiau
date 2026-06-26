# API 2 — `submit-analysis.php` (write the analysis back)

**Purpose:** the cloud Claude Code session POSTs the analysis it produced for one or more entries.
Each updated entry's `analysis_status` flips to **2** ("analyzed — awaiting review") and
`analyzed_at` is stamped.

```
POST https://perkubulve.lt/suvalgiau/submit-analysis.php?key=BRIDGE_KEY
Content-Type: application/json
```

`BRIDGE_KEY` is the value in `suvalgiau/secret.php`. Without a valid key → **403**.

## Request body

One object, or a batch under `items` (batch is preferred — one request for the whole run):

```json
{
  "items": [
    {
      "entry_id": 12,
      "calories": 540,
      "AI_description": "Avižų košė su uogomis ir kava su pienu",
      "NOVA": "3",
      "score": 72,
      "place": "Namai"
    },
    {
      "entry_id": 9,
      "calories": 620,
      "AI_description": "Mėsainis su bulvytėmis (mažesnė porcija)",
      "NOVA": "4B",
      "score": 25,
      "place": "McDonald's, Vilnius"
    }
  ]
}
```

### Fields (per item)

| field | type | notes |
|---|---|---|
| `entry_id`       | int    | **required** — the `id` from `pending.php` |
| `calories`       | int    | kcal estimate (alias accepted: `kcal`) |
| `AI_description` | text   | one short sentence in Lithuanian, what the meal was (aliases: `description`, `ai_description`) |
| `NOVA`           | text   | `"1-2"`, `"3"`, or `"4A".."4D"` (aliases: `nova`, `nova_state`) |
| `score`          | int    | **0–100 health score**, higher = healthier; values are clamped to 0–100 |
| `place`          | text   | where the meal was eaten, inferred from the description, e.g. `"Namai"`, `"McDonald's, Vilnius"` (alias: `location`; trimmed to 255 chars) |

Any missing field is stored as `NULL` for that entry. Extra/unknown fields are ignored.

## Response

```json
{ "ok": true, "updated": [12, 9], "skipped": [] }
```

- **`updated`** — entry ids that were written (they were in status 0 or 1).
- **`skipped`** — ids that were **not** written because they don't exist, or are already in
  status **2 (analyzed)** or **3 (approved)**. This guard means a re-run can't overwrite an entry
  you've already reviewed. To re-analyze a reviewed entry, reject it in the app first (→ status 1).

## Guarantees / gotchas

- **Only status 0 or 1 entries are updated.** Submitting for a status-2 or status-3 entry is a
  no-op (it lands in `skipped`).
- Send the whole run in a single request via `items` — fewer round trips.
- `score` outside 0–100 is clamped, not rejected.
- The endpoint never sets status to 3 (approved) — only you do that, in the app.

## Minimal example

```bash
curl -s -X POST "https://perkubulve.lt/suvalgiau/submit-analysis.php?key=BRIDGE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"entry_id":12,"calories":540,"AI_description":"Avižų košė su uogomis","NOVA":"3","score":72}]}'
```
