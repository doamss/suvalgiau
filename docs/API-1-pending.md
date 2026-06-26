# API 1 — `pending.php` (read the work queue)

**Purpose:** the cloud Claude Code session calls this to get every diary entry that still needs
analysis. It returns entries whose `analysis_status` is **0 (new)** or **1 (rejected, redo)**.

```
GET  https://perkubulve.lt/suvalgiau/pending.php?key=BRIDGE_KEY
```

`BRIDGE_KEY` is the value in `suvalgiau/secret.php` (`SUVALGIAU_BRIDGE_KEY`). Without a valid key
the endpoint returns **403**.

## Query parameters

| param | required | meaning |
|---|---|---|
| `key`   | yes | the bridge key |
| `user`  | no  | restrict to one user id |
| `limit` | no  | max entries to return (1–1000, default 200) |

## Response

```json
{
  "count": 2,
  "submit_to": "https://perkubulve.lt/suvalgiau/submit-analysis.php",
  "entries": [
    {
      "id": 12,
      "user_id": 3,
      "user": "Domas",
      "eaten_at": "2026-06-26 12:30:00",
      "meal_type": "lunch",
      "meal_label": "Pietūs",
      "note": "Dubuo avižų košės su uogomis ir kava su pienu",
      "photos": [
        "https://perkubulve.lt/suvalgiau/uploads/3/2026-06/ab12cd34.jpg",
        "https://perkubulve.lt/suvalgiau/uploads/3/2026-06/cd34ef56.jpg"
      ],
      "status": 0,
      "task": "analyze",
      "feedback": null,
      "previous": null
    },
    {
      "id": 9,
      "user": "Domas",
      "eaten_at": "2026-06-25 19:10:00",
      "meal_type": "dinner",
      "meal_label": "Vakarienė",
      "note": "Mėsainis su bulvytėmis",
      "photos": ["https://perkubulve.lt/suvalgiau/uploads/3/2026-06/ef56ab78.jpg"],
      "status": 1,
      "task": "revise",
      "feedback": "Porcija buvo maža, kalorijų turėtų būti mažiau.",
      "previous": { "calories": 980, "AI_description": "…", "NOVA": "4B", "score": 22, "place": "McDonald's, Vilnius" }
    }
  ]
}
```

### Field meaning

- **`status`** — `0` = new (never analyzed), `1` = you rejected the previous analysis.
- **`task`** — convenience flag: `"analyze"` for status 0, `"revise"` for status 1.
- **`feedback`** — *(status 1 only)* your free-text note describing what to fix. Honor it.
- **`previous`** — *(status 1 only)* the values you produced last time, so you can adjust rather
  than start from scratch.
- **`photos`** — array of **0 to 5** absolute image URLs for this entry; fetch/view them all to
  estimate the meal (an entry may also have text only, or several angles of the same meal).
- **`submit_to`** — the exact URL to POST your results to (see API 2).

## What the session should do

1. `GET pending.php?key=…`
2. For each entry: read `note`, view `photos`, and — if `task` is `"revise"` — apply `feedback`
   relative to `previous`.
3. Produce `{ calories, AI_description, NOVA, score, place }` for each (`place` = where it was
   eaten, inferred from the description).
4. POST them all to `submit_to` (see **API 2 — submit-analysis.php**). That flips each entry to
   status **2** ("analyzed, awaiting review").

After that, you (in the app) either **Approve** (→ status 3) or **Reject with a comment**
(→ status 1), and rejected entries come back through this endpoint on the next run.
