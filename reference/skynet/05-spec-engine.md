# 05 — The Spec Engine (`shared/spec-engine/`)

The heart of Skynet: a pure-TypeScript library converting body measurements into finished garment
specs. No I/O, no side effects; its only external inputs are two JSON files bundled at import
time (`shared/outlier-stats.json`, `shared/ease-rules.json`). It runs identically in the browser
(Home, BatchProcess) and on the server (webhook pipeline). All units are inches.

Refactored out of the old 809-line `shared/measurements.ts` in April 2026;
`shared/measurements.ts` remains as a thin back-compat shim.

## Public API

```ts
calculateSpecs(input: SpecInput): SpecOutput            // spec-engine/calculateSpecs.ts:54

SpecInput  = { gender, style, fit, bodyMeasurements: { waist, seat, thigh, knee, calf,
               fullRise, outseam, inseam?, suggestedFrontRise?, suggestedLegOpening? } }
SpecOutput = { normalizedBody, finishedSpecs, warnings: SpecWarning[],
               corrections: MeasurementCorrection[], patternReadiness: PatternReadiness }
```

- `Gender = "Men" | "Women"`, `GarmentStyle = "Straight" | "Boot" | "Fashion Boot" | "Skinny" | "Flare"`,
  `GarmentFit = "Trim" | "Regular" | "Easy"` (defined in `shared/schema.ts`, re-exported).
- `finishedSpecs.waist` is a **string range** (`"37-37.5"` = body waist to +0.5"); everything
  else numeric, plus flag fields (`thighFlag`, `calfFlag`, `frontRiseFlag`, `backRiseFlag`,
  `waistFlagSmall/Large`) and optional outlier severities/messages per measurement.

### The shim (`shared/measurements.ts:23`)

`calculateGarmentMeasurements(body, gender, style, fit, suggestedFrontRise?, suggestedLegOpening?)`
wraps `calculateSpecs` and returns
`{normalizedBodyMeasurements, garmentMeasurements, corrections, patternReadiness}`.
⚠️ **It still drops `SpecOutput.warnings`** — every current consumer (Home, BatchProcess, webhook)
re-derives flags from the boolean/severity fields instead, so the structured `SpecWarning[]` output
remains de facto unconsumed. It now **forwards `patternReadiness`** (added 2026-06), which *is*
consumed (see §11).

## Pipeline

```
normalizeBodyMeasurements → gender dispatch → style/fit formulas → applyThighRangeCorrection
→ getThighRange + classifyThighFlag → applyRise → computeOutliers → finishedSpecs
→ generateWarnings → generatePatternReadiness → SpecOutput
```

### 1. Normalization (calculateSpecs.ts:21-52)

- inseam (if >0) quarter-rounded; missing outseam ← `roundQuarter(inseam − 0.5)`; missing
  inseam ← `roundQuarter(outseam + 0.5)`.
- missing calf ← `knee − 1`; missing knee ← `calf + 1` (⚠️ these two are NOT quarter-rounded).
- Gender dispatch is `gender === "Women" ? womens : mens` — any unexpected string silently runs
  the men's path.

`roundQuarter(v) = Math.round(v*4)/4` is used pervasively (half-up: 0.125 → 0.25).

### 2. Men's formulas (mens.ts:16-163)

Base: `lowerWaist = roundQuarter(body.waist)`; `baseOutseam = inseam ? inseam − 0.5 : outseam`
(inseam takes priority when both supplied).

**Skinny** (ignores fit; always +0.5" ease):
seat/thigh/calf = body + 0.5; knee = (calf ≤ knee ? knee : calf) + 0.5;
legOpening = max(calf − 4, 0); outseam = base − 0.5.

**Non-Skinny ease by fit:**

| Fit | seat | thigh | knee (base) |
|---|---|---|---|
| Trim | +1.5 | +0.5 | +1.5 |
| Regular | +2.0 | +1.0 | +2.0 |
| Easy | +3.0 | +2.0 | +3.0 |

**Straight knee/calf** (mens.ts:68-85): if `bodyCalf ≥ bodyKnee || (knee − calf) < 0.5`
("too similar"): calf = bodyCalf + (kneeEase − 0.5); naturalKnee = calf + 0.5;
knee = max(naturalKnee, **17**); if the 17" minimum forced knee up, recompute calf = knee − 1
(this is the April 2026 "calf bug" fix). Otherwise: knee = max(bodyKnee + kneeEase, **17**);
calf = knee − 1. *The 17" knee minimum applies to men's Straight only.*

**Boot**: knee = bodyKnee + kneeEase; calf tapering: knee ≤ 18 → calf = knee;
≤ 21 → knee − 0.5; > 21 → knee − 1.

**Fashion Boot**: if bodyCalf ≥ bodyKnee: calf = bodyCalf + (kneeEase − 0.5), knee = calf + 0.5;
else knee = bodyKnee + kneeEase, calf = knee − 0.5.

**Leg opening & outseam** (mens.ts:109-123):

| Style | legOpening | outseam |
|---|---|---|
| Boot | calf < 18 → max(calf, **17.5**); calf ≤ 20 → **18**; else **18.5** | base **+ 1** |
| Fashion Boot | calf ≤ 20 → clamp(calf − 3, 16, 17); else 18.5 | base |
| Straight | max(calf − (calf ≥ 19 ? 3.0 : 2.5), **14**) | base |

⚠️ **Men's Flare is a silent hybrid**: mens.ts has no Flare branch — Men+Flare falls into the
Fashion Boot `else` for knee/calf but the Straight `else` for legOpening/outseam. The path is
reachable (webhook and clients normalize 'flare'→'Flare' for men).

### 3. Women's formulas (womens.ts:206-303)

**Fit ease** (note: can be negative):

| Fit | seat | thigh | knee | calf |
|---|---|---|---|---|
| Trim | **−1.5** | **−1.0** | −0.5 | −0.5 |
| Regular | 0 | 0 | 0 | 0 |
| Easy | +1.0 | +1.0 | +0.5 | +0.5 |

kneeEase/calfEase feed only the Straight path; Boot/Flare use `adaptiveKneeEase(bodyKnee,
calfDiff, fit)` (womens.ts:22-33), which picks 0.5–1.5" based on bodyKnee brackets (<15, <17,
=17, else) and fit.

**Skinny** (no ease): if calf ≥ knee → both forced to bodyKnee and **calfFlag = true**;
if knee−calf ≤ 0.25 → calf = knee − 0.5; else passthrough. legOpening = max(calf − 4, **9.5**);
outseam = base − 0.5.

**Straight**: same "too similar" rule as men's; too-similar → calf = bodyCalf + 1.5 + calfEase,
knee = calf + 0.5; normal → knee = bodyKnee + 2 + kneeEase, calf = knee − 1.
legOpening = calf − (calf ≤ 15 ? 2 : calf < 18 ? 2.5 : 3), no floor. outseam = base.

**Boot** (womens.ts:108-167): if calfDiff > 0.5 → knee/calf = body + 1, legOpening = calf +
(calfDiff ≥ 1 ? 1 : 0.5). Else: target legOpening from bodyKnee brackets (16.5 / 17.5 / 18 / 19),
knee = bodyKnee + adaptiveKneeEase, then gap-based calf/legOpening resolution.
Final floor: legOpening ≥ **17.5**; outseam = base **+ 2**.

**Flare** (womens.ts:169-202): target legOpening anchored to fit-adjusted thigh, clamped
**21–26**; knee = bodyKnee + adaptiveKneeEase; gap ≤ 4.5 → calf = knee + 1.5; else knee + 2;
legOpening from remaining gap. outseam = base **+ 2**.

**Fallback** (womens.ts:258-264): any other style (e.g. Women + "Fashion Boot"):
knee = bodyKnee + 2, calf = knee − 1, legOpening = **14**, outseam = base.

### 4. Thigh-range correction (corrections.ts) — the auto-correction system

`applyThighRangeCorrection({gender, finishedSeat, thighCandidate})` (corrections.ts:268-317)
checks the candidate against `getThighRange(finishedSeat, gender)`:

- **In range** → unchanged, no correction.
- **Too small** (both genders): thigh raised to exactly thighMin. No downward tolerance ever.
- **Too large — Men** (corrections.ts:45-149), stepwise until pass:
  1. thigh −0.5 (pass if ≤ max, no tolerance)
  2. seat +0.5 (pass if thigh ≤ max **+ 0.25**)
  3. thigh −1.0 total (pass if ≤ max(seat+0.5) + 0.25)
  4. thigh capped at −1.0; loop seat +0.5 until thigh ≤ max + 0.25
- **Too large — Women** (corrections.ts:153-257): thigh −0.5 → thigh −1.0 (still no tolerance,
  seat unchanged) → seat +0.5 (+0.25 tolerance) → loop seat +0.5.

**Key invariant**: the 0.25" tolerance applies ONLY after the seat has been raised. Men raise
seat at step 2; women at step 3. Every correction emits one `MeasurementCorrection`
(`id: "thigh_range_correction"`, `method: "thigh_range_adjustment"`, `reviewRequired: true`,
originalValues = the pre-correction *finished* values, not raw body values).

Because corrections normalize out-of-range thighs first, `thighFlag` / `THIGH_OUT_OF_RANGE` is
effectively only reachable within the 0.25" tolerance window.

### 5. Thigh range lookup (lookupTables.ts)

- `mensSeatThighTable` (:5-15): seat 39 → [21,22] … 80 → [37.5,45.5].
- `womensSeatThighTable` (:26-51): seat 35.5 → [19.5,20.5] … 56 → [28.75,?]; minimums
  pattern-maker-confirmed (April 2026) or interpolated; maximums are P95 from a 10,496-pattern
  women's knowledge base (per-seat n documented in code comments). Seats 51 and 53 are marked
  "data P5 (unconfirmed by pattern maker)".
- `getThighRange(finishedSeat, gender)` linearly interpolates between bracketing keys —
  and **extrapolates** beyond the table edges using the first/last two entries.

### 6. Rise (rises.ts + lookupTables.ts:99-122)

`calculateFrontBackRise`: nominal split front **11.25** / back **15.75** (= 41.67%/58.33% of a
27" nominal total), scaled by `fullRise / 27`. Ranges = scaled ± 0.25.
⚠️ The `isWomens` parameter is a **no-op** — both ternaries return the same values for both
genders.

`applyRise(fullRise, isWomens, suggestedFrontRise?)`:
- With a suggested front rise (from Asana `Front Rise` field or UI): frontRise = rounded
  suggestion; backRise = fullRise − suggestion; `frontRiseFlag` set if the (unrounded)
  suggestion falls outside the ±0.25 proportional range.
- Without: backRise = rounded proportional back; frontRise = fullRise − backRise (front absorbs
  rounding remainder); flags false.

### 7. Outlier detection (outliers.ts)

`computeOutliers(gender, style, correctedSeat, lowerWaist, correctedThigh, knee, calf,
finalLegOpening, fullRise)` computes five metrics and checks each against
`outlier-stats.json[gender][style].ratios`:

| Metric | Formula | Thresholds |
|---|---|---|
| seatToWaistDiff | seat − waist (inches) | outside range99 (±3σ) → **error**; outside range95 (±2σ) → **warning** |
| kneeToThigh | knee/thigh × 100 | same ladder |
| calfToKnee | calf/knee × 100 | same ladder |
| legOpenToThigh | legOpening/thigh × 100 | same ladder |
| fullRiseToSeat | fullRise/seat × 100 | **asymmetric**: lower = mean − 3σ, upper = mean + 2σ; both produce **warning only**, never error |

- Missing gender/style stats → silently `'normal'` for all checks. **No Men/Flare and no
  Women/Fashion Boot entries exist** — those combos get zero statistical checks.
- ⚠️ The `confidence === 'insufficient'` short-circuit at outliers.ts:34 is **dead code**:
  `confidence` is destructured from the per-ratio object but the JSON stores it at the style
  level, so it's always `undefined`.
- The kneeToThigh result is reported under the `knee` field (there is no THIGH_OUTLIER code).
- Callers pass the corrected seat/thigh and the final (possibly overridden) leg opening.

### 8. Waist handling

Finished waist is always the string `"{waist}-{waist+0.5}"` (`formatWaistRange`, utils.ts:39-43).
Flags: `waistFlagSmall` when seat − waist < 3"; `waistFlagLarge` when > 9".

### 9. Warnings (warnings.ts) — currently unconsumed

`generateWarnings(specs)` emits coded `SpecWarning`s (THIGH_OUT_OF_RANGE,
CALF_UNUSUAL_PROPORTION, WAIST_TOO_CLOSE_TO_SEAT, WAIST_TOO_FAR_FROM_SEAT,
FRONT/BACK_RISE_OUT_OF_RANGE, plus *_OUTLIER passthroughs). The shim drops them and no caller
reads them — they exist for future consumers (Optitex, human review queues).

### 10. Ease JSON (ease.ts) — dead code

`getEase` + `shared/ease-rules.json` (439 historical patterns) are exported but **never called**
by any calculation path. The live ease values are the hardcoded tables above. If ever wired up:
only STRAIGHT (n=385), FASHBOOT (n=17), BOOT (n=23) pass the ≥15-sample gate; SKINNY (n=1) and
the missing FLARE key fall back to STRAIGHT medians.

### 11. Pattern readiness (patternReadiness.ts) — added 2026-06, and consumed

`generatePatternReadiness(ctx)` is the final pipeline step (calculateSpecs.ts). Unlike
`generateWarnings`, its output **is** consumed: the shim forwards it, `ResultsDisplay` renders a
"Pattern Guidance" card and posts it, the Asana comment writer builds a "Pattern Guidance" comment
section from it (server/asana.ts), and it is persisted to `optitex_jobs.pattern_readiness`.

It receives a `PatternReadinessContext` of `{ bodyMeasurements: normalizedBody, finishedSpecs,
corrections, warnings }` and returns:

```ts
PatternReadiness = {
  status: "ready_for_pattern" | "needs_review" | "blocked",
  summary: string,
  reasons: string[],            // blocking issues (status === "blocked")
  reviewPoints: string[],       // patternmaker attention items (status === "needs_review")
  adjustmentsApplied: string[], // correction `reason` strings
}
```

**Decision rules** (in order):

- **blocked** — only genuine "can't draft" conditions: no leg length (both `outseam` and `inseam`
  ≤ 0) **or** `fullRise` missing/≤ 0. Intended to be rare.
- **needs_review** — any of: `seat − waist < 1"` (note: a *different* threshold from the engine's
  own 3"/9" waist flags in §8); `waistFlagLarge`/`waistFlagSmall`; `thighFlag`; `calfFlag`;
  `frontRiseFlag`/`backRiseFlag`; **any** applied correction; **any** outlier — warning *or* error.
- **ready_for_pattern** — none of the above.

⚠️ `adjustmentsApplied` is built from `corrections.map(c => c.reason)` for `blocked`/`needs_review`
but returned `[]` in the `ready_for_pattern` branch — harmless today (corrections always force
`needs_review`), but a latent inconsistency. The `warnings` context field is passed in but unused;
readiness is derived from `finishedSpecs` flags/outliers and `corrections`.

Exposed via the shim and the `spec-engine/index.ts` barrel (`generatePatternReadiness`,
`PatternReadinessContext`, `PatternReadiness`, `PatternReadinessStatus`). New `calculateSpecs.test.ts`
cases cover ready_for_pattern, needs_review (corrections / seat−waist<1), and blocked (no leg length).

## Overrides from Asana/UI

- `suggestedLegOpening` → `resolveLegOpening` replaces the calculated leg opening (truthy check —
  an explicit 0 is ignored).
- `suggestedFrontRise` → rise override with range flagging (see §6).

## Tests (calculateSpecs.test.ts)

Plain script with throwing asserts — no jest/vitest; run with
`npx tsx shared/spec-engine/calculateSpecs.test.ts`. Covers: men's Straight normal/too-similar/
17"-minimum cases (including the April 2026 calf-bug regression), men's Boot tapering + floors,
women's Skinny zero-ease + calfFlag, women's Straight Trim with correction, inseam→outseam
derivation, suggestedLegOpening override, correction-supersedes-warning, and 14 unit cases for
the men's/women's correction sequences against exact table points. ⚠️ tsconfig excludes
`**/*.test.ts`, so `npm run check` never type-checks this file, and no CI runs it.

## Magic numbers cheat-sheet

17" men's-Straight knee minimum · 17.5" Boot leg-opening floor (both genders) · 9.5" women's
Skinny leg-opening floor · 14" men's Straight leg-opening floor · 21–26" women's Flare
leg-opening clamp · 3"/9" waist-seat flag thresholds · 0.25" correction tolerance (post-seat-raise
only) · 0.5" waist range width · 11.25/15.75 rise nominals · outseam offsets: Skinny −0.5,
men's Boot +1, women's Boot/Flare +2.
