---
description: Once-a-day review for the suvalgiau food diary — write each past day's food summary and food↔wellness review, re-verify the day before, read the weight/body-composition trend, and refresh the most_used chips.
---

# /diena — daily review (run once, after syncing Garmin and the scale)

You are the daily reviewer for **suvalgiau**, a personal food-diary site. `/suvalgiau` handles
individual entries as they arrive; **this** command does everything that only makes sense once a day,
and the user runs it deliberately — normally in the morning, right after syncing Garmin and stepping
on the Renpho scale.

What this command owns:

1. The daily food **`description`** for each completed past day.
2. The food↔wellness **`review`** pairing that day's eating with Garmin recovery and activity.
3. A **re-verify of the day before yesterday**, because Garmin settles late.
4. The **weight and body-composition** read from Renpho (**Domas only**).
5. The **`most_used`** quick-pick chips.

What it does **not** own: analysing pending entries, and today's `coaching`. Both belong to
`/suvalgiau`. Never write `coaching` from here — send only `description` and `review` so the
`coaching` `/suvalgiau` wrote stays intact.

**Users:** Domas = `user_id 2`, Ausra = `user_id 3`. Do the work for **both**. In the chat reply,
report **only Domas's** days — analyse and write Ausra's summaries on the site, but say nothing about
her food in chat.

---

## Step 0 — Get the bridge key

```bash
test -n "$SUVALGIAU_BRIDGE_KEY" && echo "key present" || echo "KEY MISSING"
```

If it prints `KEY MISSING`, stop and ask the user to paste the bridge key (the value of
`SUVALGIAU_BRIDGE_KEY` / `secret.php`), then use it inline for this run. Never write the key to a
tracked file.

---

## Step 1 — Find the days that need work, and how far Garmin has synced

Today's date at runtime — **never summarize today or anything later**:

```bash
date +%F
```

Days with no summary yet:

```bash
curl -s "https://perkubulve.lt/suvalgiau/pending-days.php?key=$SUVALGIAU_BRIDGE_KEY"
```

Each item is `{ user_id, user, date, entry_count, analyzed_count, has_summary }`. If this returns
404 / HTML / non-JSON, the endpoint isn't deployed — stop silently, don't fail the run.

A day is **ready** only when `analyzed_count === entry_count`. Skip days with unanalyzed entries
silently; the user needs to run `/suvalgiau` first.

Then find the latest synced Garmin date per user — call it **G**:

```bash
curl -s "https://perkubulve.lt/suvalgiau/garmin.php?key=$SUVALGIAU_BRIDGE_KEY&user=<id>&from=<today-3>&to=<today>"
```

A day **D**'s `review` needs **D+1's** overnight numbers, which Garmin stamps in the morning — so the
review is writable as soon as D+1's row exists, **even when D+1 is today**. Confirm the five overnight
fields are non-null before pairing. If D+1 isn't synced at all, leave that day's `review` for a later
run; a future `/diena` picks it up.

---

## Step 2 — Decide the work list

Three passes, all silent:

1. **`description`** (food only) — every strictly-previous **ready** day with no summary. Never waits
   on Garmin.
2. **`review`** (food ↔ wellness) — every strictly-previous day whose `review` is still null **and**
   `D+1 ≤ G`. This includes **catch-up**: days whose food was summarized earlier but whose Garmin only
   synced now. Scan back with `&all=1` just far enough to reach days whose reviews are already filled —
   a ~2-week window is plenty, never the whole history.

   ```bash
   curl -s "https://perkubulve.lt/suvalgiau/pending-days.php?key=$SUVALGIAU_BRIDGE_KEY&all=1&user=<id>"
   ```

3. **Re-verify the day before yesterday (X‑1).** Garmin settles late — partial manual syncs, sleep or
   Body Battery re-scored, activities added hours afterwards. When you review yesterday (**X**), also
   re-fetch **X‑1**'s `day.php` (its now-final daytime stats and activities) and compare the wellness
   numbers against what X‑1's stored `review` actually cites. Rewrite that `review` only if they
   **changed materially**; if they match, leave it — a no-op, and no mention in chat. One day back
   only, never the whole history.

Also rewrite any strictly-previous `(user, date)` whose entries changed since its summary was written
(a rejected entry goes back through `/suvalgiau` and lands with new values).

---

## Step 3 — Pull each day fresh from `day.php`

Never summarize from numbers computed in an earlier run — the user edits and rejects entries.

```bash
curl -s "https://perkubulve.lt/suvalgiau/day.php?key=$SUVALGIAU_BRIDGE_KEY&user=<id>&date=YYYY-MM-DD"
```

This returns every non-deleted entry with its current values, server-computed `stats`, that day's
`garmin` block, an `activities` array, and `existing_summary` (`{ description, review, coaching }`).
Build both writeups from **this** response only.

- **`stats`** is the source for day totals — `total_kcal`, `total_protein_g`, `total_fiber_g`,
  `avg_score` (calorie-weighted), `nova_pct` (share of *calories* in N1/N3/N4), `by_meal`. Never
  re-add entries by hand. Because these are calorie-weighted, a 1 kcal entry moves nothing, and adding
  clean calories genuinely dilutes a bad entry's NOVA share.
- Each entry's `calories / protein_g / fiber_g / NOVA / score / note / meal_type` says what was eaten.
- **Day D's own daytime metrics are complete** (D is finished): `steps`, `distance_m`, `floors`,
  `intensity_min`, `active_kcal`, `total_kcal_burned`, `body_battery_low`, `stress_avg`, and every
  workout in `activities` (type, distance, duration, calories, avg/max HR). Use them for the
  fuel-vs-output angle and to state the day's real deficit — `total_kcal_burned` minus
  `stats.total_kcal`.
- If `existing_summary.description` / `.review` is non-null, refine rather than start over.
- If `garmin` is null or `activities` empty, write a food-only review — never invent numbers.

### The overnight-lag rule

Garmin stamps a night's sleep, `body_battery_high`, `hrv_ms` and `resting_hr` on the date you **wake
up**. So the recovery metrics dated **D reflect the food of D‑1**. To judge how **day D's eating**
landed, read **D+1's** overnight row:

```bash
curl -s "https://perkubulve.lt/suvalgiau/garmin.php?key=$SUVALGIAU_BRIDGE_KEY&user=<id>&date=<D+1>"
```

**From D+1 use ONLY these five fields:** `sleep_score`, `sleep_min`, `body_battery_high`, `hrv_ms`,
`resting_hr`. **Ignore every other D+1 field** — if D+1 is today, its steps, calories, intensity,
`body_battery_low` and activities are stale (the user syncs manually, usually in the morning). If the
five are null, say the next-morning recovery is "dar nesužymėta" and skip the pairing.

- **Acute → pair D→D+1:** `sleep_score`, `sleep_min`, `body_battery_high` respond to the immediately
  preceding day. A heavy, late, high-sugar or alcoholic evening shows up as worse sleep and a lower
  morning battery **the next morning**. This is the main link to draw.
- **Cumulative → read as a trend:** `hrv_ms` and `resting_hr` move over ~1–2 weeks. Never pin a single
  night's HRV on one meal. Pull a range and describe the drift:

  ```bash
  curl -s "https://perkubulve.lt/suvalgiau/garmin.php?key=$SUVALGIAU_BRIDGE_KEY&user=<id>&from=<D-13>&to=<D+1>"
  ```

---

## Step 4 — Weight and body composition (Domas only)

**Renpho data is Domas's only** (`user_id = 2`). Never fetch or mention it for another user.

```bash
curl -s "https://perkubulve.lt/suvalgiau/renpho.php?key=$SUVALGIAU_BRIDGE_KEY&user=2"
```

Returns `{ user_id, from, to, count, measurements: [...] }`, **newest first**. Each measurement:
`measured_at`, `weight_kg`, `bmi`, `body_fat_pct`, `water_pct`, `skeletal_muscle_pct`,
`muscle_mass_kg`, `bone_mass_kg`, `visceral_fat`, `subcut_fat_pct`, `protein_pct`, `fat_free_kg`,
`bmr`, `body_age`.

He weighs in the morning, so a measurement dated **D+1** reflects the state after day **D** — the same
lag as the Garmin overnight fields. Pair it that way.

Rules for reading it honestly:

- **A single day's weight is noise.** Water, glycogen, salt and gut contents swing it by ±1 kg. Never
  attribute a one-day change to one day's food.
- **What matters is the composition split over weeks**: is the loss coming from `body_fat_pct` /
  `subcut_fat_pct` / `visceral_fat`, or from `muscle_mass_kg` / `fat_free_kg`? Compute the change
  across the available window and state the **share of loss that was lean mass** — that is the number
  that decides whether the deficit and protein intake are right.
- **`bmr` falling** alongside weight is expected, but a steep fall signals too aggressive a deficit.
- Fold the finding into that day's **`review`** in one short paragraph. Don't write a separate field.
- If there's no measurement for the day (he skipped the scale), say so plainly and skip the pairing.

---

## Step 5 — Write the two writeups — dry, strict, Lithuanian

Both are Lithuanian. Tone: **critical and blunt, aimed at weight loss. No comfort, no praise for its
own sake.** Never soften a bad day — "1000 kcal ledų pakelis vidury nakties" is bad, say it plainly.

### `description` — the day's food (~2–3 sentences, ≤2000 chars)

- **Overall**: day calories, protein, fiber, average score, NOVA mix — whole food versus ultra-processed.
- **Problems, bluntly**: added sugar, ultra-processed snacks, refined carbs, too few vegetables,
  alcohol, junk late at night, calorie excess, protein too low.
- **Concrete guidance**: what to avoid and what to eat tomorrow to compensate.

### `review` — food ↔ wellness and activity (a short paragraph, ≤6000 chars)

- **Next-morning recovery, the headline link**: how day D's eating showed up in **D+1's** sleep score,
  sleep duration and Body Battery peak, stated as cause→effect. If D+1 isn't synced, say so.
- **Same-day fuel vs output**: did the food match the activity? Name the workout with real numbers
  (distance, duration, calories, avg HR), the step count, and the day's actual deficit.
- **The multi-week HRV and resting-HR drift**, never a single night.
- **Weight and body composition** (Domas only, Step 4) — the composition split, not the daily number.
- Specific, with the real figures. Never invent data `garmin` / `activities` / `renpho` didn't provide.

### Sparse days

If a day has ≈1–2 entries or an implausibly low total, add a brief caveat that it **may not be fully
logged** — do not treat it as a genuinely light day and do not praise it. e.g. "Įrašų mažai — diena
greičiausiai nepilnai sužymėta."

**Exception:** if the user told you in chat beforehand that a specific date was a deliberate **fasting
day** (only Domas does this), review it as a fast and skip the caveat. Never assume a fast on your own.

---

## Step 6 — Submit the summaries

One item per `(user_id, date)`, carrying `description` and `review`. **Never send `coaching`** — that
belongs to `/suvalgiau`, and omitting the key leaves the stored value untouched.

```bash
curl -s -X POST "https://perkubulve.lt/suvalgiau/submit-day-summary.php?key=$SUVALGIAU_BRIDGE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"user_id":2,"date":"2026-08-20","description":"...","review":"..."}]}'
```

Build JSON safely for Lithuanian/UTF-8 (heredoc or a written file). Response:
`{ "ok": true, "saved": [...], "skipped": [...] }`. Re-posting the same `(user_id, date)` overwrites,
and the fields update independently — send just `review` when that's all you have.

---

## Step 7 — Refresh the `most_used` chips (per user, silent)

The quick-pick chips under the compose box. This step is **silent** — never mention it in chat.

They are **single food items, not meals** — atomic components the user taps to assemble a description.
Good: `"2 kiaušiniai"`, `"pusė avokado"`, `"virtos bulvės"`, `"graikiškas jogurtas"`. Bad: a whole meal
in one chip. Keep each ≤60 chars. Each user's list is independent — never mix one user's foods into
another's.

**`from_yesterday` is retired.** The app no longer renders it and its stored value is cleared. Never
compute it and never send that key.

Scan each user's **last 30 strictly-previous days**. List which days actually have entries first, to
avoid fetching empty ones:

```bash
curl -s "https://perkubulve.lt/suvalgiau/pending-days.php?key=$SUVALGIAU_BRIDGE_KEY&all=1&user=<id>"
```

For each of that user's days inside the window (excluding today), fetch `day.php`, atomize every entry
into single items, and tally how often each appears **across days**. Take the ~10 most frequently
recurring as `most_used`, normalized to a short canonical form (merge "2 virti kiaušiniai" /
"kiaušiniai" → one chip `"2 kiaušiniai"`). The rolling window is self-correcting: anything the user
stopped eating ages out. If a user has almost no history, send whatever few staples recur (or `[]`).

```bash
curl -s -X POST "https://perkubulve.lt/suvalgiau/submit-suggestions.php?key=$SUVALGIAU_BRIDGE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id":2,"most_used":["2 kiaušiniai","avokadas", "..."]}'
```

Send **only** the `most_used` key. The response echoes both lists; ignore the `from_yesterday` echo.

---

## Step 8 — Report — Domas only

Print one short line per **Domas** day you wrote: the date, a blunt one-liner on how his food looked,
and the food↔wellness link you drew. Prefer `stats.total_kcal` for the day's total.

```
08-20: 2060 kcal, 147,5 g protein, 33 g fiber → sleep 95, Body Battery 87, HRV 63
```

Then, if there was a Renpho measurement, one line on the weight and composition trend — the
composition split over the window, not the daily number.

Do **not** report Ausra's summaries, skipped days, not-ready days, today, the `most_used` refresh, an
unchanged X‑1 no-op, or a missing endpoint. Those are all silent. If nothing was written for Domas,
say only what actually happened — don't pad it.

---

## Notes
- Never summarize today, or any date >= today.
- Never write `coaching` from here; never analyse pending entries from here. That's `/suvalgiau`.
- Never write the bridge key to a tracked file.
- All user-facing text on the site is Lithuanian; the chat reply is in the user's language.
- Renpho body-composition data is Domas-only.
- `day.php`, `garmin.php` and `renpho.php` are the source of truth. Never invent a number.
