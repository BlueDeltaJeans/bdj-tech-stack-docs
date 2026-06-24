# 04 — Asana Integration

> **📍 `file:line` citations may be stale.** `asana.ts` (and `routes.ts`) have **grown since this
> doc was written**; the `asana.ts:NNN` / `routes.ts:NNN` numbers below can be off by roughly
> **50–300 lines**. **Search by symbol name** (e.g. `postFinishedMeasurements`,
> `parseAsanaMeasurements`, `registerAsanaWebhook`) instead of jumping to the line number.

All Asana communication lives in `server/asana.ts` and uses the **raw REST API**
(`https://app.asana.com/api/1.0`) with native `fetch`. The `asana` npm package in package.json is
**dead** (nothing imports it; `server/asana.d.ts` is a leftover type shim), as is
`@replit/connectors-sdk` — both are residue from the pre-April-2026 OAuth connector approach.

## Authentication

`asanaRequest(path, method, body?)` (asana.ts:4-57, private) reads the PAT from
`process.env.ASANA_ACCESS_TOKEN` per request and sends `Authorization: Bearer <token>`. Throws
"ASANA_ACCESS_TOKEN is not configured" if absent. Friendly error mapping: 403 → permission
denied, 404 → task deleted, 429 → rate limit.

## Exported functions (asana.ts)

| Function | Line | Purpose |
|---|---|---|
| `fetchAsanaTask(taskId)` | :383 | GET task with `TASK_OPT_FIELDS` (gid, name, created_at, memberships+project names, custom_fields with name/display/number/text/enum values) |
| `fetchAsanaTaskForWorkOrder(taskId)` | :388 | Same plus notes, due_on, tags |
| `parseAsanaMeasurements(task)` | :273 | Custom fields → `ParsedMeasurements` (see below). Always returns an object (`{format:'unknown'}` at minimum) |
| `postFinishedMeasurements(taskId, m)` | :464 | Builds + posts the results comment (see below) |
| `hasFinishedMeasurements(taskId)` | :433 | Dedupe: scans task stories for `'auto-calculated by garment measurement calculator'` OR `'finished measurements'` OR (waist AND seat AND knee AND front-rise keyword pairs). ⚠️ Returns `false` on any API error — fails open |
| `searchTasksByName(q, limit=20)` | :524 | Workspace typeahead search. Numeric queries also search `#<q>` (matches names like "#101378 Customer"). Uses the FIRST workspace from `/workspaces`, cached forever in a module variable |
| `registerAsanaWebhook(projectGid, targetUrl)` | :97 | POST /webhooks with filters for task `added` AND `changed` (even though the processor only handles `added`) |
| `parseVTInputs(task)` | (VT section) | Reads the Virtual Tailor input fields by display name: `Gender`, `Age`, `Weight`, `VT Height`, `VT Shoe Size`, `VT Waist`, `VT Inseam`, `VT Bra Size`. Maps gender to `'m'`/`'w'`; `hasVTFields` is true only when a VT-specific field has a value (Gender/Age/Weight exist on all orders) |
| `setAsanaCustomFields(taskId, fieldsByGid)` | (VT section) | Skynet's first custom-field WRITE capability: `PUT /tasks/:id` with `{data:{custom_fields:{gid: stringValue}}}`. Requires the PAT to have write access |
| `writeBoldMetricsMeasurements(taskId, m)` | (VT section) | Writes the six Bold Metrics outputs by GID (`BOLD_METRICS_FIELD_GIDS`): Waist Avg. / Hip Circum / Thigh Circum / Knee Circum / U Crotch / Jean Inseam |
| `listAsanaWebhooks()` / `deleteAsanaWebhook(gid)` | :119/:151 | Webhook management |
| `listAsanaProjects()` | :76 | All projects across all workspaces |
| `getAsanaApiInstances()` | :59 | Back-compat shim exposing `storiesApi` for routes.ts |

## Measurement parsing — `parseAsanaMeasurements` (asana.ts:273-376)

Fields are matched by **exact display name** (never GID). The em-dash `'—'` display value is
treated as empty. Value precedence: `number_value` → `text_value` → `display_value` →
`enum_value.name` (asana.ts:216-226).

**Format detection** (asana.ts:280-286): MANUAL if `Waist Around` or `Seat Around` present;
else BOLD_METRICS if `Waist Avg.` or `Hip Circum` present. Manual wins if both exist.

### Manual format (tape measurements) — values used as-is, NO rounding

| Engine field | Asana custom field |
|---|---|
| waist | `Waist Around` |
| seat | `Seat Around` |
| thigh | `Thigh Upper Around` |
| knee | `Knee Around` |
| calf | `Calf Around` |
| fullRise | `Full Rise` |
| outseam | `Outseam` |
| suggestedFrontRise | `Front Rise` (optional) |
| suggestedLegOpening | `Leg Opening` (optional) |

### Bold Metrics format ("Virtual Tailor") — EVERY value rounded to nearest 0.25"

`roundQuarter(v) = Math.round(v*4)/4` (asana.ts:212-214).

| Engine field | Asana custom field | Notes |
|---|---|---|
| waist | `Waist Avg.` | Note the trailing period. From Bold Metrics, this field holds the **average** of the three returned waist circumferences (preferred/natural/stomach) |
| seat | `Hip Circum` | |
| thigh | `Thigh Circum` | |
| knee | `Knee Circum` | |
| fullRise | `U Crotch` | |
| inseam | `Jean Inseam` | when present, derived `outseam = roundQuarter(inseam − 0.5)` |
| calf | `Calf` | if absent, derived `calf = roundQuarter(knee − 1)` |

### Note-field fallback

If neither format matches, the `Note` custom field is regex-parsed (`Waist Avg: X`,
`Hip Circum: X`, etc., asana.ts:240-271) with the same Bold Metrics rounding and derivations.

### Demographics (all formats)

- **gender** ← `Gender` field, raw ("W", "M", "Women", "Men", …) — normalized later by callers.
- **style** ← `Style` or `Garment Style`, via `normalizeStyleValue` (asana.ts:228-238):
  uppercase + strip whitespace, then `FASHBOOT`/`FASHIONBOOT`→Fashion Boot, `BOOT`→Boot,
  `STRAIGHT`/`STRT`→Straight, `SKINNY`/`SKNY`→Skinny, `FLARE`→Flare; unknown passes through.
- **fit** ← `Fit` or `Garment Fit`: Tight→Trim, Tailored→Regular, Relaxed→Easy. Default when
  missing: gender W/Women→Trim, M/Men→Regular. **Skinny style always forces Trim**
  (duplicated rule: asana.ts:373 and routes.ts:1086).

## Posting results — `postFinishedMeasurements` (asana.ts:464-510)

One Asana story (comment) per task:

```
**FINISHED MEASUREMENTS**
Waist: 37-37.5"
Seat: 44"
Thigh: 25"
Knee: 18"
Calf: 17"
Leg Opening: 14.5"
Front Rise: 11.5"
Back Rise: 16"
Inseam: 31.5"            ← Inseam if measurements.inseam truthy, else Outseam

**⚠️ MEASUREMENT ALERTS**          ← only when at least one flag exists
⚠️ Knee: <kneeOutlierMessage>
⚠️ Thigh: This measurement is too large for this style
🔧 Correction: <reason>            ← one line per corrections[] entry (wrench = auto-correction)

**🧭 PATTERN GUIDANCE**            ← when patternReadiness is supplied (status + summary +
   Status / Summary / Review Points / Blocking Issues / Adjustments Applied lists)

—
*Auto-calculated by Garment Measurement Calculator*
```

⚠️ The footer text is what `hasFinishedMeasurements` keys on for dedupe — changing the comment
format risks double-posting. The webhook pipeline now forwards `corrections` (🔧 lines appear on
all three posting paths) and, for Virtual Tailor orders Skynet computed itself, an optional
`⚠️ Bold Metrics: …` line carrying Bold Metrics' own outlier flag.

> **⚠️ Pattern Guidance is client-only.** The **🧭 PATTERN GUIDANCE** block renders only when
> `measurements.patternReadiness` is supplied. The **webhook automation path never supplies it** —
> its `finishedMeasurements` object in `processWebhookEvent` (`server/routes.ts`) omits
> `patternReadiness`, so auto-processed orders get the measurements + 🔧 corrections (+ optional
> Bold Metrics) comment but **no Pattern Guidance section**. Only the client paths
> (`POST /api/asana/task/:id/measurements`) include it. See docs 01 and 03.

## Webhook lifecycle

1. **Register** (Automation UI → `POST /api/asana/webhooks/register {projectGid}`): server
   computes target `https://${REPLIT_DOMAINS[0]}/api/asana/webhook` and calls Asana. Asana
   immediately POSTs a handshake containing `X-Hook-Secret`.
2. **Handshake** (routes.ts:648-668): secret persisted into `webhook_settings.webhook_secret`,
   echoed back. On registration success the server stamps `webhookRegisteredAt` so catch-up
   events for pre-existing tasks are skipped.
3. **Events**: HMAC-SHA256 over the raw body, verified against the stored secret, then async
   processing (full pipeline in doc 03). Only `action === 'added'` events do anything; `changed`
   events (which the registration also subscribes to) just inflate `totalEventsProcessed`.

## Client-side Asana components

- **AsanaTaskLookup.tsx** — search-and-load combobox on the Calculator page: 300ms debounce,
  ≥2 chars, `GET /api/asana/search` then `GET /api/asana/task/:id`; rejects `format:'unknown'`
  with a toast; otherwise feeds measurements into the form.
- **AsanaTaskSelector.tsx** — slimmer search-only variant (returns gid+name, no fetch). Used
  **only inside ResultsDisplay** ("Select Asana Task" card) to pick a posting target when
  measurements were entered manually. Near-duplicate of the lookup combobox.
- **Automation.tsx** — webhook dashboard (see docs 07 and 09 for its broken orders query and
  cosmetic project labels).

## Gotchas specific to this integration

- Custom-field renames in Asana silently break parsing (name-based matching, including the
  trailing period in `Waist Avg.`).
- Single-workspace assumption: `getWorkspaceGid` caches the first workspace forever.
- Quarter-rounding applies ONLY to Bold Metrics values; manual tape values pass through raw.
- Batch name-resolution's first-result fallback can process the wrong task.
- The webhook registration subscribes to `changed` events that are always discarded — harmless
  but noisy, and the reason the retry endpoint silently does nothing.
