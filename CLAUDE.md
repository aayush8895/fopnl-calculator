# fopnl-cacluclator — Nutrient Level Checker

Single self-contained file: `nutrient-checker.html`. No build step, no server, no
dependencies. Distributed by handing someone the file — they open it directly
(`file://`) in a mobile or desktop browser.

## What it does

Classifies a packaged food's Saturated Fat, Added Sugars, and Sodium as
Low/Medium/High per FSS (Labelling & Display) Regulation, 2020, using the RDA
table the user supplied:

- RDA: Saturated fat 22g, Added sugars 50g, Sodium 2000mg.
- Threshold row depends on **serving size**, not just %RDA in isolation:
  - ≤30g serving: Low ≤5%, Medium 6–19%, High ≥20%
  - 31–199g serving: Low ≤10%, Medium 11–29%, High ≥30%
  - ≥200g serving: Low ≤15%, Medium 16–39%, High ≥40%
- %RDA is rounded to the nearest **integer** before banding — rounding to 1
  decimal first leaves gaps (e.g. 5.4% falls in no band). Bands must be applied
  to integers to stay exhaustive over all real inputs.

Two ways to get numbers in: manual typing, or a label photo read by the
Gemini API (Google AI Studio). Photo extraction is optional — the calculator
works standalone with typed values.

## Key design decisions / gotchas (read before touching the extraction code)

**Per-serving vs per-100g is not a formatting detail — it changes which
threshold row applies.** A 12.5g-serving product that only prints per-100g
grams must not be scaled into the wrong bucket. The extraction prompt/schema
asks the model for `perServing` and `per100g` as **separate objects** plus a
`servingSizeG`, and the app computes the actual per-serving amount locally:
prefer a directly-printed per-serving value; otherwise scale
`per100g × servingSize/100`; otherwise leave null (never default to 0 — a
missing value must never silently read as "Low").

**Real Indian labels often print "Per 100 g (% RDA per serve — Xg)"** — i.e.
per-100g gram values with the %RDA-for-the-serving shown in parentheses right
next to them. This *looks* like per-serving data at a glance but isn't. This
tripped up an early version of the UI (not the extraction — the extraction
was already correct): the values landed in the collapsed "Per 100g" section
and the visible "Per serving" boxes stayed empty, which read as "extraction
failed." Fixed in `applyExtractedFields()`: when no per-serving gram value
was printed, it now auto-computes and fills the visible per-serving boxes
immediately, auto-expands the Per-100g `<details>`, and — important — the
result panel's "Source:" line says *"no per-serving value printed —
calculated from per-100g × serving size"* rather than falsely claiming
"as printed." A `dataset.autocalc` flag on the `ps_*` inputs tracks this
provenance and is cleared the moment the user edits the field by hand.

**Sodium has its own fallback chain** (mg direct → salt g × 400 → per-100g mg
scaled → per-100g salt scaled). Salt→sodium conversion is `sodium_mg =
salt_g × 400` (salt÷2.5, then ×1000). Total Sugars is never used to infer
Added Sugars — if the label only prints Total Sugars, Added Sugars stays null
and the UI says so explicitly.

**Extraction request shape:** uses `generationConfig.responseMimeType:
"application/json"` + `responseSchema` (officially supported structured
output), with `inline_data`/`mime_type` (snake_case) for the image part —
that field casing is what the REST API actually expects, live-verified.
An earlier draft copied a "prime the reply with a fake trailing `role:
'model'` turn saying `{`" trick from a different project's server-side
integration (`billing-app`) — that shape is flatly rejected by current
models on this account: `400 Requests ending with a model turn are not
supported.` Don't reintroduce it.

**Model selection:** default is `gemini-3.5-flash`, editable in Settings. The
"Test key & find models" button hits `GET /v1beta/models`, filters to
Gemini/Gemma models that support `generateContent` and aren't
embedding/tts/audio/video/image-gen/etc., ranks flash > pro, latest > versioned,
deprioritizes preview/exp, and repopulates the model dropdown — this is how
the app self-heals as Google's model lineup changes instead of relying on a
hardcoded name staying valid.

**Key storage:** localStorage only, namespaced (`nlc_gemini_api_key`,
`nlc_gemini_model`, `nlc_shortcut_dismissed`), wrapped in try/catch (some
`file://`/private-mode configs throw on localStorage access — must not
white-screen the page). The key never leaves the browser except in direct
calls to `generativelanguage.googleapis.com`.

**First-run UX:** a dismissible banner with OS-specific desktop-shortcut
instructions (no `beforeinstallprompt`/manifest — that's dead on `file://`).

## Testing this file (no test framework — do it like this)

There's no CI, no npm project here. To verify changes:

1. **Pure logic** (classification bands, null-safety, salt/sodium math, JSON
   brace-balancing parser): extract the relevant functions into a throwaway
   Node script and assert against hand-computed values, especially the
   boundary integers (5/6%, 19/20%, 29/30%, 39/40% for each of the three
   serving-size buckets).
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
     click/fill it — this is the actual cause of two different "timeout"
     failures during development, not app bugs.
   - Prefer `page.evaluate` to set input values + dispatch an `input` event
     over `page.fill()` when the value is a secret (an API key) — a `fill()`
     that times out dumps the literal value into Playwright's retry log,
     which lands in whatever is capturing the command output.
4. **Live API calls:** a real Gemini key is needed to validate the actual
   request/response shape (mocking this out would miss exactly the kind of
   bug that showed up: the model-priming 400, and transient 503s under
   demand). A working key lives in `../billing-app/config.json`
   (`geminiApiKey` field) — that's a different project's production key, so
   ask before reusing it, and treat any accidental appearance of it in logs
   /transcripts as a reason to rotate it at aistudio.google.com/apikey.

## Non-goals

No backend, no build tooling, no package.json — keep it that way. The whole
point is "hand someone one file." Resist adding a bundler, a framework, or a
server component even for convenience.
