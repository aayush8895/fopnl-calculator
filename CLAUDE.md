# fopnl-cacluclator — Nutrient Level Checker

Single self-contained file: `nutrient-checker.html`. No build step, no server, no
dependencies, no package.json. Distributed by handing someone the file — they
open it directly (`file://`) in a mobile or desktop browser.

Repo: https://github.com/aayush8895/fopnl-calculator (branch `main`).

## Current scope: basic calculator only

The file is currently a **plain calculator with no AI, no photo capture, and
no first-run UI** — those were deliberately stripped out to test the core
calculation logic in isolation first (see "Previously removed" below if
re-adding them later). What's actually in the file today:

- A serving-size field, three "per serving" nutrient fields (Sat. fat g,
  Added sugars g, Sodium mg), and a collapsible "per 100g" section with the
  same three plus two salt fields (g, per-serving and per-100g).
- A Calculate button that classifies each of the three nutrients as
  Low/Medium/High.
- Bidirectional live sync between fields (see below) so the user doesn't
  have to do the per-100g ↔ per-serving math by hand.

## Classification logic

RDA: Saturated fat 22g, Added sugars 50g, Sodium 2000mg (FSS Labelling &
Display Regulation, 2020). Threshold row depends on **serving size**:

- ≤30g serving: Low ≤5%, Medium 6–19%, High ≥20%
- 31–199g serving: Low ≤10%, Medium 11–29%, High ≥30%
- ≥200g serving: Low ≤15%, Medium 16–39%, High ≥40%

%RDA is rounded to the nearest **integer** before banding — rounding to 1
decimal first leaves gaps (e.g. 5.4% falls in no band). Bands must be applied
to integers to stay exhaustive over all real inputs (verified for every
integer 0–45 across all three buckets).

Per-serving vs per-100g is not a formatting detail — it changes which
threshold row applies, so the two are never conflated. `resolveNutrient()`
prefers a directly-entered per-serving value, falls back to scaling
`per100g × servingSize/100`, and otherwise returns null — **null is never
treated as 0** (a missing value must never silently read as "Low").
`resolveSodium()` has an extra fallback rung: mg direct → salt g × 400
(salt→sodium: ÷2.5 then ×1000) → per-100g mg scaled → per-100g salt scaled.

## Live field sync (the main interactive feature)

Two independent sync groups, both driven by `input` listeners set via plain
`.value =` assignment (never `dispatchEvent`, so there's no feedback-loop risk
— setting a field's value via JS doesn't fire its own `input` listener):

1. **Sat fat and Added sugars** (`PAIRS` array): a simple 2-way link per
   nutrient — `ps_*` ↔ `p100_*`, converted through the serving size
   (`per100 = perServing × 100/serving`). Clearing either side clears both
   (a stale, silently-wrong number in the other field is worse than losing
   an in-progress edit — this was a real bug caught by testing, see below).

2. **Sodium/salt — four fields, one underlying quantity** (`SODIUM_META` /
   `syncSodiumSalt()`): `ps_sodium` (mg), `p100_sodium` (mg), `ps_salt` (g),
   `p100_salt` (g). Editing *any one* of the four recomputes the other
   three: same-scale conversion (mg ↔ g, ×400 or ÷400) never needs serving
   size; crossing serving↔100g does. This is a fuller sync than the
   sat-fat/sugar pairs — the user's own framing was "so that everything is
   in sync," and sodium has two units *and* two scales, so a simple pair
   link isn't enough coverage.

3. **Filling in serving size retroactively** (after nutrient fields already
   have values) backfills both groups — per-serving takes priority as the
   source when both sides already happen to have data, matching
   `resolveNutrient`'s own priority order. For the sodium group, whichever
   of the four fields has a value first (in `SODIUM_IDS` order) becomes the
   sync source.

**Bug already found and fixed once:** an early version of the sat-fat/sugar
sync didn't propagate a *clear* (only non-empty edits), so blanking a
per-100g field left a stale per-serving number that Calculate would still
treat as freshly-entered, valid data. Both sync groups now clear their
partner(s) when any one field is emptied. If you touch this code again,
re-run the clear-propagation case specifically — it's easy to reintroduce.

## Testing this file (no test framework — do it like this)

There's no CI, no npm project here. To verify changes:

1. **Pure logic** (classification bands, null-safety, salt/sodium math):
   extract the relevant functions into a throwaway Node script and assert
   against hand-computed values, especially the boundary integers (5/6%,
   19/20%, 29/30%, 39/40% for each of the three serving-size buckets).
2. **DOM wiring / ids:** `grep -o "getElementById('[^']*')"` vs `grep -o
   'id="[^"]*"'` catches typo'd element ids fast.
3. **Full browser E2E:** no browser tool is bundled in this environment, but
   Playwright + a cached Chromium build exist on this machine from other
   projects. Load Playwright via `createRequire('/home/aayush/pprojects/
   tacoza/package.json')('playwright')` (don't add a package.json here just
   for testing), and launch with `executablePath:
   '/home/aayush/.cache/ms-playwright/chromium-1234/chrome-linux64/chrome'`
   (the version Playwright expects by default, chromium-1228, isn't the one
   actually cached — override it explicitly). `headless: true, args:
   ['--no-sandbox']`.
   - Any element inside a collapsed `<details>` must be opened
     (`page.evaluate(() => el.open = true)`) before Playwright can
     click/fill it — this caused two false "timeout" failures during
     development that looked like app bugs and weren't.
   - Use `page.fill()` freely here — there's no API key/secret in this
     version of the file, so no risk of a timeout retry-log leaking one
     (that risk applies if AI extraction gets re-added; see below).

## Previously removed (not preserved in git history — this repo's first
commit is the stripped-down version)

An earlier iteration of this file also had: an optional camera/file-picker
photo capture that read nutrient values off a label photo via the Gemini API
(Google AI Studio), a Settings panel for caching the API key in
localStorage and picking/ranking a model, and a first-run "add a desktop
shortcut" banner. All of it was removed on request to validate the base
calculator first — none of that code is retrievable from `git log` since it
predates this repo's initial commit. If asked to re-add it, the key lessons
learned (worth not re-discovering the hard way):

- Use `generationConfig.responseMimeType: "application/json"` +
  `responseSchema` for structured extraction, with `inline_data`/`mime_type`
  (snake_case) for the image part. Do **not** use a "prime the reply with a
  fake trailing `role: 'model'` turn saying `{`" trick — that shape is
  flatly rejected by current models on this account:
  `400 Requests ending with a model turn are not supported.`
- Model selection should hit `GET /v1beta/models`, filter to Gemini/Gemma
  models supporting `generateContent` (excluding embedding/tts/audio/video/
  image-gen), rank flash > pro and latest > versioned, deprioritize
  preview/exp, and let that repopulate a model dropdown — don't hardcode a
  model name, it goes stale as Google's lineup changes.
- Real Indian labels often print "Per 100 g (% RDA per serve — Xg)" — gram
  values that are per-100g with the %RDA-for-the-serving shown alongside,
  which looks like per-serving data at a glance but isn't. Extraction should
  ask for `perServing` and `per100g` as separate objects, and the UI should
  auto-compute + clearly label ("no per-serving value printed — calculated
  from per-100g × serving size") rather than leave the visible per-serving
  boxes empty, which reads as "extraction failed" when it didn't.
- A working Gemini key for live-testing lives in `../billing-app/config.json`
  (`geminiApiKey` field) — a different project's production key; ask before
  reusing it, and prefer `page.evaluate` to set it + dispatch an `input`
  event over `page.fill()`, since a timed-out `fill()` dumps the literal
  value into Playwright's retry log.

## Non-goals

No backend, no build tooling, no package.json — keep it that way. The whole
point is "hand someone one file." Resist adding a bundler, a framework, or a
server component even for convenience.
