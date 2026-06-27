---
description: Fetch pending food-diary entries, analyze each (calories, NOVA, health score, dry verdict), and submit results back to the suvalgiau site.
---

# /suvalgiau — analyze pending food-diary entries

You are the analysis engine for **suvalgiau**, a personal food-diary site. The user writes a short
description and/or uploads photos of what they ate. Your job: pull every pending entry, estimate and
judge it, and write the results back. The user then approves or rejects each result in the app.

Run mode is **auto-submit all**: fetch → analyze every entry → POST all results in one batch →
print a summary. The user reviews in the app afterward.

---

## Step 0 — Get the bridge key

The key lives in the environment variable `SUVALGIAU_BRIDGE_KEY`. Resolve it:

```bash
test -n "$SUVALGIAU_BRIDGE_KEY" && echo "key present" || echo "KEY MISSING"
```

If it prints `KEY MISSING`, stop and ask the user to paste the bridge key (the value of
`SUVALGIAU_BRIDGE_KEY` / `secret.php`), then use it inline for this run. Do **not** write the key to
any tracked file. Recommend they set it as an environment variable in their Claude Code web
environment config so future runs work without pasting.

---

## Step 1 — Fetch the work queue

```bash
curl -s "https://perkubulve.lt/suvalgiau/pending.php?key=$SUVALGIAU_BRIDGE_KEY"
```

Returns JSON: `{ count, submit_to, entries: [...] }`. Each entry has:

- `id` — the `entry_id` you submit results for.
- `note` — free-text description (may be empty). May contain hints like the venue ("normaliam
  kavinukėj" / "pigioj kavinėj" / "namuose").
- `photos[]` — 0–5 absolute image URLs. **View every photo** to estimate the meal.
- `meal_type` / `meal_label` — breakfast/lunch/dinner/snack.
- `status` — `0` = new, `1` = you rejected the previous analysis (redo).
- `task` — `"analyze"` (status 0) or `"revise"` (status 1).
- `feedback` — *(revise only)* the user's note on what to fix. **Honor it.**
- `previous` — *(revise only)* the values you produced last time `{calories, AI_description, NOVA,
  score, place}`. Adjust relative to these rather than starting over.

If `count` is 0, report "Nieko nelaukia analizei" and stop.

---

## Step 2 — View the photos

For each entry with photos, download and view them so the visual estimate is real, not guessed:

```bash
# example — save each photo, then Read it as an image
curl -s "<photo_url>" -o /tmp/suvalgiau_<entry_id>_<n>.jpg
```

Then use the Read tool on each saved file to actually see the food. Multiple photos of one entry are
usually angles of the same meal — don't double-count portions.

---

## Step 3 — Determine NOVA via the Nuodai API

NOVA is the ultra-processing classification: `1-2` (whole / minimally processed, good), `3`
(processed), `4A`–`4D` (ultra-processed, worsening severity). The Nuodai API is public, read-only,
no key:

```bash
# find a product id
curl -s "https://perkubulve.lt/nuodai/api.php?action=list&q=<lithuanian+product+name>"
# full detail: NOVA verdict + every ingredient + the NOVA-4 drivers
curl -s "https://perkubulve.lt/nuodai/api.php?action=product&id=<id>"
```

Use `rating.code` for the NOVA code. Lithuanian letters are folded in search (`ž→z`), every query
word must appear in the name.

**How to assign NOVA to a diary entry (which is a *meal*, not a single barcode):**

- **Packaged / branded product** (a named drink, snack, bar, sauce, ready meal) → look it up and use
  its `rating.code` directly.
- **Cooked / restaurant meal** → judge by what dominates and how processed the venue is. The user
  signals venue in the note:
  - "namuose" / home-cooked from whole ingredients → usually `1-2` or `3`.
  - "normali kavinė" (normal/decent café) → typically `3`; ultra-processed components push toward
    `4A`–`4B`.
  - "pigi kavinė" (cheap café), fast food, fried, heavy sauces → `4B`–`4D`.
  - **All emulsified/industrial sauces (mayonnaise, ketchup, dressings), processed meats (sausages,
    bacon, deli), sweetened/soft drinks, packaged sweets and snacks → NOVA 4.** If a meal centers on
    these, the meal is NOVA 4. Look up the specific sauce/product in Nuodai to pin the sub-grade
    (4A–4D) when you can.
- Report the single NOVA code that best represents the meal as eaten. When a meal mixes a whole-food
  base with a NOVA-4 component, grade up toward the processed component (the user wants the more
  negative read).
- If you truly cannot tell and there's no usable signal, prefer the worse plausible grade rather than
  assuming clean.

---

## Step 3.5 — When a shop or café is named, look the product up online

If the note names **where** the item came from — a shop ("iš Maximos", "Rimi", "Lidl", "Iki") or a
specific café/bakery ("bandelė iš X kavinės") — don't just estimate. **Find the real product online**
to get accurate calories / ingredients / portion, then base your numbers on that.

Lookup order (stop as soon as you have solid data):

1. **Nuodai API** (Step 3) — covers Barbora + Rimi + lastmile groceries; try it first for packaged items.
2. **The named retailer's own site** — these publish nutrition + ingredients per product:
   - **Maxima** → `maxima.lt` (its confectionery/bakery products list ingredients and kcal).
   - **Rimi** → `rimi.lt`. **Lidl** → `lidl.lt`. **Iki** → `iki.lt`. **Barbora** → `barbora.lt`.
   - Use `WebSearch` for `"<product name> <shop> kcal"` or `site:maxima.lt <product>`, then `WebFetch`
     the product page and pull the **per-100 g** kcal + ingredients (and weight, to get the portion total).
3. **The café/bakery's own site or menu** — search the café name + dish; many publish nutrition or at
   least portion weight. For a generic café `bandelė`/pastry with no published data, search a
   comparable product (e.g. "cinamono bandelė kcal") and use that, erring high.

From whatever you find:
- Use the page's **per-100 g kcal** × the portion weight for `calories` (still err high when the
  portion is uncertain).
- Use the **ingredient list** to set `NOVA` — emulsifiers, glaze, margarine, additives, syrups →
  NOVA 4; cross-check the additive in the Nuodai product endpoint when useful.
- Note the source in your chat summary (e.g. "kcal iš maxima.lt") so the user can sanity-check.

If nothing usable is online, fall back to a photo/portion estimate and **say so** in the summary.

---

## Step 4 — Produce the four judgements per entry

### `calories` (int, kcal)
If a shop/café was named, prefer the **real online figure** (Step 3.5: retailer per-100 g kcal ×
portion weight). Otherwise estimate from photos + note + portion cues. **When ambiguous, err high** —
pick the upper end of the plausible range. The user would rather reject an over-estimate than under-count. For `revise` entries,
move in the direction `feedback` asks (e.g. "porcija buvo maža" → lower from `previous.calories`).

### `AI_description` (short Lithuanian sentence)
A **dry, blunt verdict in Lithuanian**: what the meal was, and whether it's good or bad food, what to
avoid. Not cheerful. Examples of tone:
- `"Avižų košė su uogomis — pilnavertis, skaidulų ir lėtų angliavandenių pagrindas, geras pasirinkimas."`
- `"Mėsainis su bulvytėmis — ultra-perdirbtas greitas maistas, daug riebalų ir druskos, vengti."`
- `"Saldus gazuotas gėrimas — tuščios kalorijos, jokios maistinės vertės, atsisakyti."`
Keep it one sentence, factual, slightly harsh when deserved.

### `NOVA` (text)
The code from Step 3: `"1-2"`, `"3"`, or `"4A"`/`"4B"`/`"4C"`/`"4D"`.

### `score` (int 0–100, health score, higher = healthier)
0 = junk that would be better thrown away; 100 = excellent food actively improving health, full of
what the body needs. Anchor it:

- **Start from NOVA**: NOVA 1–2 → ceiling ~100; NOVA 3 → ceiling ~70; NOVA 4 → ceiling ~40 (4A) down
  to ~10 (4D).
- **Then adjust within that band** for nutrient quality:
  - **Up**: vegetables, fruit, legumes, whole grains, lean protein, fish, fiber, healthy fats (olive
    oil, nuts), no added sugar.
  - **Down**: added/refined sugar, refined flour, deep-fried, processed/red meat excess, high
    saturated fat, high salt, alcohol, empty calories.
- Calibration anchors: deep-fried fast food + soda ≈ 5–15; greasy café meal ≈ 25–40; decent mixed
  restaurant plate ≈ 45–60; balanced home-cooked meal with veg + protein ≈ 70–85; salad/veg + lean
  protein + whole grain, no junk ≈ 85–100.
- Values are clamped 0–100 server-side, but stay in range.

### `place` (text)
Where it was eaten, inferred from the note ("Namai", "McDonald's, Vilnius", a café name, etc.). If the
note gives no venue, infer the best you can (e.g. "Namai" for obviously home food) or use a generic
label like "Kavinė". Trimmed to 255 chars.

---

## Step 5 — Submit the whole batch

One POST with all entries under `items`:

```bash
curl -s -X POST "https://perkubulve.lt/suvalgiau/submit-analysis.php?key=$SUVALGIAU_BRIDGE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"items":[
    {"entry_id":12,"calories":540,"AI_description":"...","NOVA":"3","score":72,"place":"Namai"},
    {"entry_id":9,"calories":620,"AI_description":"...","NOVA":"4B","score":25,"place":"McDonald'\''s, Vilnius"}
  ]}'
```

Build the JSON safely (a heredoc or a written file is fine for Lithuanian/UTF-8 and apostrophes —
avoid breaking the shell quoting). Response: `{ "ok": true, "updated": [...], "skipped": [...] }`.

- `updated` = entries written (now status 2, awaiting your review).
- `skipped` = entries already in status 2/3 or nonexistent — a no-op guard, not an error.

---

## Step 6 — Report

The queue is **multi-user** — each entry carries `user_id` and `user` (display name), so a single run
may mix several people's meals (e.g. "Domas" and "Ausra"). Submitting is unaffected: `entry_id` is
globally unique. Show **whose** entry each row is.

Print a compact summary table of what was submitted, e.g.:

| id | user | place | kcal | NOVA | score | verdict |
|----|------|-------|------|------|-------|---------|

Then state the `updated` / `skipped` ids from the response. Mention any entry you found hard to
estimate so the user knows where to look when reviewing in the app.

To analyze just one person, pass their id to `pending.php` via `&user=<id>` (optional — by default
process everyone's pending entries).

---

## Step 7 — Daily summaries for previous days (NEVER today)

After the per-entry work above, also write a **daily summary** for each completed *past* day that
doesn't have one yet. This is a strict, weight-loss-oriented review of the whole day — not a per-meal
note.

### 7a. Find days that need a summary

```bash
curl -s "https://perkubulve.lt/suvalgiau/pending-days.php?key=$SUVALGIAU_BRIDGE_KEY"
```

Default (`all=0`) returns only days **without** a summary ("empty days") — **scan only these unless
the user explicitly asks to re-summarize** (then add `&all=1`). Each item:
`{ user_id, user, date, entry_count, analyzed_count, has_summary }`.

- If this returns a 404 / HTML / non-JSON, the endpoint isn't deployed yet → **skip Step 7 entirely
  and say so once**. Don't fail the run.

### 7b. Decide which days to summarize

- **NEVER summarize today.** Get today's date at runtime (`date +%F`) and skip any day whose `date`
  is **>= today**. Only strictly-previous days.
- Only summarize a day that is **ready**: `analyzed_count === entry_count` (every entry that day is
  analyzed). Skip days with unanalyzed entries and mention them.
- Multi-user: a day is per `(user_id, date)` — handle each user's day separately.

### 7c. Get that day's meals

You need the day's entries (calories, AI_description, NOVA, score, meal_type, note) to review them.
Sources, in order:
1. Entries you analyzed **earlier in this same run** for that date — reuse them.
2. A per-entry fetch for that `(user, date)` that includes **analyzed** entries, if the backend
   supports it (e.g. `pending.php` with a date/status filter that returns status 2–3).

If you genuinely cannot retrieve a day's meals, **skip that day and report it** — never invent a
summary from the counts alone.

### 7d. Write the summary (`description`) — dry, strict, Lithuanian

Tone: **critical and blunt, aimed at weight loss. No comfort, no praise for the sake of it.** Do not
soften bad days ("1000 kcal ledų pakelis vidury nakties" is bad — say it plainly). Cover:

- **Overall**: total/approx day calories, balance, NOVA mix (how much ultra-processed vs whole food),
  junk vs real meals.
- **Problems, bluntly**: e.g. too much sugar, ultra-processed snacks, refined carbs (white bread,
  rice, chips), no/too few vegetables, alcohol, eating junk late, calorie excess.
- **Concrete guidance**: what to **avoid** and what to **eat tomorrow** to compensate (e.g. "rytoj —
  daugiau daržovių ir baltymų, jokių saldumynų, traškučių ir saldžių gėrimų").
- Keep it ~2–3 sentences (UI panel is small; trimmed to 2000 chars). Strict, factual, useful.

### 7e. Submit (batch)

```bash
curl -s -X POST "https://perkubulve.lt/suvalgiau/submit-day-summary.php?key=$SUVALGIAU_BRIDGE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"user_id":2,"date":"2026-06-26","description":"..."}]}'
```

One item per `(user_id, date)`. Build JSON safely for Lithuanian/UTF-8 (heredoc/file). Response:
`{ "ok": true, "saved": [...], "skipped": [...] }`. Re-posting the same `(user_id, date)` overwrites.

### 7f. Report

List which day summaries you wrote (per user + date), and which past days you skipped and why
(not ready / meals unavailable / endpoint missing).

---

## Notes
- Re-runs are safe: the server won't overwrite entries you've already approved/analyzed (status 2/3).
  To re-analyze, reject it in the app first (→ status 1) and it returns to the queue.
- Never write the bridge key to a tracked file.
- Keep all user-facing food text in Lithuanian; keep the chat summary in the user's language.
