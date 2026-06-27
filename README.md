# suvalgiau — food-diary analysis bridge

Personal food diary on [perkubulve.lt/suvalgiau](https://perkubulve.lt/suvalgiau). You log a meal
(text description and/or up to 5 photos); a Claude Code session pulls the pending entries, estimates
and judges each one, and writes the result back. You then **Approve** or **Reject (with a comment)**
each result in the app.

## The `/suvalgiau` command

Run `/suvalgiau` in a Claude Code session. It:

1. Reads the bridge key from `$SUVALGIAU_BRIDGE_KEY`.
2. `GET pending.php` — every entry in status **0** (new) or **1** (rejected → redo).
3. For each entry: views the photos, reads the note, applies your feedback (on redos), and looks up
   NOVA scores via the public Nuodai API. **If you name a shop or café** (e.g. "bandelė iš Maximos"),
   it searches that retailer's site (maxima.lt, rimi.lt, lidl.lt, iki.lt…) for the product's real
   calories and ingredients instead of guessing.
4. Produces `{ calories, AI_description, NOVA, score, place }` per entry.
5. `POST submit-analysis.php` — one batch. Each entry flips to status **2** (awaiting your review).
6. **Daily summaries:** checks `pending-days.php` for completed *past* days without a summary (never
   today), writes a strict, weight-loss-oriented review of the whole day, and posts it via
   `submit-day-summary.php`.
7. Prints a summary.

### What it estimates

| field | meaning |
|---|---|
| `calories` | kcal estimate. **Errs high when ambiguous** — reject + comment to correct. |
| `AI_description` | dry, blunt verdict in **Lithuanian** — what it was, good/bad, what to avoid. |
| `NOVA` | ultra-processing class: `1-2` / `3` / `4A`–`4D`, via the Nuodai dataset. |
| `score` | 0–100 health score. 0 = junk; 100 = excellent, health-improving food. |
| `place` | venue, inferred from the note. |

## Setup

Set the bridge key as an environment variable in your Claude Code web environment config so every
session can use it:

```
SUVALGIAU_BRIDGE_KEY = <the value of secret.php / SUVALGIAU_BRIDGE_KEY>
```

The key is **never** committed. If it's missing, `/suvalgiau` asks you to paste it for that run only.

## APIs

The site exposes three HTTP APIs (full reference in [`docs/`](docs/)):

- [`docs/API-1-pending.md`](docs/API-1-pending.md) — read the work queue (`pending.php`, keyed).
- [`docs/API-2-submit-analysis.md`](docs/API-2-submit-analysis.md) — write results back
  (`submit-analysis.php`, keyed).
- [`docs/API-3-nuodai-nova.md`](docs/API-3-nuodai-nova.md) — public NOVA / ultra-processing lookup
  over ~33k Lithuanian grocery products (`nuodai/api.php`, no key).
- [`docs/API-4-pending-days.md`](docs/API-4-pending-days.md) — which days still need a daily summary
  (`pending-days.php`, keyed).
- [`docs/API-5-submit-day-summary.md`](docs/API-5-submit-day-summary.md) — write a day's summary
  (`submit-day-summary.php`, keyed).

## Tuning the analysis

The scoring rubric, calorie bias, NOVA assignment rules, and the tone of `AI_description` live in
[`.claude/commands/suvalgiau.md`](.claude/commands/suvalgiau.md). Edit that file to change how meals
are judged.
