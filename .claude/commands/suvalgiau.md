---
description: Fetch pending food-diary entries, analyze each (calories, NOVA, health score, dry verdict), and submit results back to the suvalgiau site.
---

# /suvalgiau — analyze pending food-diary entries

You are the analysis engine for **suvalgiau**, a personal food-diary site. The user writes a short
description and/or uploads photos of what they ate. Your job: pull every pending entry, estimate and
judge it, and write the results back. The user then approves or rejects each result in the app.

Run mode is **auto-submit all**: fetch → analyze every entry → POST all results in one batch →
print a summary. The user reviews in the app afterward.

**Scope: this command is per-entry only.** It runs many times a day, so it does only the work that new
entries create — analyse them, submit them, refresh **today's** `coaching`, and report where Domas's
day stands. Everything that runs **once a day** lives in **`/diena`**: the daily food `description`,
the food↔wellness `review`, **past days'** `coaching`, the two-days-ago re-verify, the weight and
body-composition read, and the `most_used` chips. Never check whether those are due from here.

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

**Home-cooked dishes — judge by ingredients, don't over-grade.** NOVA classifies by *type of
processing*, not by who cooked it or whether additives *could* exist. Plain staples are **NOVA 1**
(minimally processed): flour, plain pasta, rolled oats, rice, fresh curd, eggs, milk, plain meat,
vegetables, fruit. Butter, oil, salt, sugar are **NOVA 2** (culinary ingredients). So for a dish the
user clearly cooked from scratch:
  - All whole foods + basic kitchen staples (e.g. curd + egg + flour + salt + butter) → **NOVA 1‑2**
    ("minimally processed home cooking"). Do **not** bump it to 3 just because flour or salt is
    present — flour alone is NOVA 1.
  - Bump to **NOVA 3** only if a genuinely *processed* component is part of it (shop bread, cheese,
    a cured/smoked-meat accent, canned/jarred processed food).
  - Reserve **NOVA 4** for an actual industrial/additive-laden component (sausage/deli meat,
    emulsified sauce, hydrogenated-oil marinade, packaged mix, glaze, syrup).
  The "err negative" lean is for genuine ambiguity — not a reason to up-grade clean home cooking.

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
Where it was eaten, inferred **only from this entry's own note** ("Namai", "McDonald's, Vilnius", a
café name, etc.). If the note gives no venue, use a generic label ("Namai" for obviously home food,
else "Kavinė"/"Restoranas"). Trimmed to 255 chars.

**Never infer a place from another user's entries.** Each person's location is independent — do NOT
put one user at a venue just because another user was there at a similar time (e.g. don't place Ausra
at "Vilnius Outlet" because Domas logged it). Cross-user context is off-limits for `place`. If a
user's own note gives no venue, use a generic label rather than borrowing someone else's.

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

**Chat commentary is only about the user in chat (Domas, `user_id = 2`).** You still analyze and
submit every user's entries (and write each user's day summary on the site), but in the chat reply do
**not** add coaching/editorial commentary about other users' food (e.g. Ausra) — just list their rows
in the table. Save any "what to cut / how the day looked" nudges for Domas's own entries.

To analyze just one person, pass their id to `pending.php` via `&user=<id>` (optional — by default
process everyone's pending entries).

**Then report Domas's own day status in chat** (`user_id = 2`), from his `day.php` `stats` for today:
total calories, protein, fiber, average score and NOVA mix so far, plus what is still open — how much
protein is left to reach ~130 g and roughly how many calories of room remain. Quote this morning's
`sleep_score` / `sleep_min` / `body_battery_high` / `hrv_ms` / `resting_hr` when they add something.
**Never quote today's Garmin step/calorie/activity counters** — the user syncs manually in the morning,
so for the current day those are stale and understate the day (see Step 7).

---

## Step 7 — Refresh today's `coaching` (per user, silent except for Domas)

The **only** day-level write this command makes is **today's** `coaching` field. Everything else that
runs once a day — the daily `description`, the food↔wellness `review`, **past days'** `coaching`, the
two-days-ago re-verify, the weight/body-composition read and the `most_used` chips — lives in
**`/diena`**. Do **not** call `pending-days.php` here, do not look for past days that need a summary,
do not rewrite any `coaching` dated before today, and do not touch `most_used`. If a past day needs
work, the user runs `/diena`.

For **each user who has entries today**, pull the day fresh and rewrite `coaching`:

```bash
curl -s "https://perkubulve.lt/suvalgiau/day.php?key=$SUVALGIAU_BRIDGE_KEY&user=<id>&date=$(date +%F)"
```

Use `stats` for the day's numbers (`total_kcal`, `total_protein_g`, `total_fiber_g`, `avg_score`,
`nova_pct`, `by_meal`) — server-computed, matches the app, never re-add entries by hand.

**⚠️ Today's Garmin daytime counters are stale and must be ignored.** The user syncs Garmin manually,
usually in the morning, so for the **current day** `steps`, `distance_m`, `floors`, `intensity_min`,
`active_kcal`, `total_kcal_burned`, `body_battery_low` and `activities` reflect only the hours up to
that sync — never the whole day. Never quote them, never compute a deficit from them, never call the
day sedentary because of them.

**Today's overnight fields are fine** — Garmin stamps `sleep_score`, `sleep_min`,
`body_battery_high`, `hrv_ms` and `resting_hr` in the morning and they are final once synced. Use
those freely: they describe how last night went, which is exactly the feedback the day starts from.

`coaching` is Lithuanian, blunt, weight-loss-oriented, and **forward-looking** — it is advice for the
rest of today, not a post-mortem. Cover:

- **Where the day stands**: calories, protein, fiber, average score, NOVA mix so far.
- **What the best and worst entries were**, and specifically what made them so.
- **What is still needed before bed**: how much protein is left to reach ~130 g, roughly how many
  calories of room remain, and a concrete suggestion or two that fits.
- **This morning's recovery numbers** and what they imply for today (train / go easy / sleep earlier).
- Any standing issue worth one line (e.g. resistance training still not started).

Re-posting the same `(user_id, date)` overwrites, and the fields update independently — send **only**
`coaching` so the stored `description` and `review` are left untouched:

```bash
curl -s -X POST "https://perkubulve.lt/suvalgiau/submit-day-summary.php?key=$SUVALGIAU_BRIDGE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"items":[{"user_id":2,"date":"2026-08-21","coaching":"..."}]}'
```

Write it for every user with entries today; say nothing in chat about anyone but Domas.

---

## Notes
- Re-runs are safe: the server won't overwrite entries you've already approved/analyzed (status 2/3).
  To re-analyze, reject it in the app first (→ status 1) and it returns to the queue.
- Never write the bridge key to a tracked file.
- **Terminology in the chat reply.** `varškė` is **curd** in English, not "quark" — curd is the drier,
  granular Lithuanian product (Žemaitijos, Dvaro, Pilos, Amfora). Reserve "quark" for products actually
  sold as quark/kvarg (e.g. Lindahls). Same for `grietinė` = sour cream, `varškės sūris` = curd cheese.
- **Language is split by destination, always.** Everything written to the site — `AI_description`,
  `coaching`, `description`, `review`, `most_used` — is **Lithuanian**. Everything written in the chat
  reply is **English**, including table headers, table cells, food names and figures. Never mix
  Lithuanian words or phrases into the chat reply; quote a stored Lithuanian line only when the user
  explicitly asks to see what was written to the site.
- Today's Garmin step/distance/calorie/activity counters are stale (manual morning sync) — never quote
  them or compute a deficit from them. Today's overnight sleep/battery/HRV/resting-HR fields are final
  and fine to use.
- Daily work (`description`, `review`, past days' `coaching`, X‑1 re-verify, weight review,
  `most_used`) belongs to `/diena`. This command writes only today's `coaching`.
