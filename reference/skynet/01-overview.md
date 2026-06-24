# 01 — Overview & End-to-End Flows

## What Skynet does

Pattern makers need *finished garment* measurements (what the jeans should measure) derived from
*body* measurements (what the customer measures). Skynet:

1. **Pulls body measurements from Asana order tasks** — either manual tape measurements or
   Bold Metrics "Virtual Tailor" estimates, parsed from custom fields by name.
2. **Calculates finished garment specs** using gender/style/fit-specific ease formulas in
   `shared/spec-engine/`.
3. **Validates everything**: thigh-range checks against seat-indexed lookup tables, automatic
   thigh/seat corrections, waist-to-seat sanity flags, rise-range flags, and statistical outlier
   detection against ~5,300 historical patterns (95%/99% confidence intervals).
4. **Posts the results back to the Asana task** as a `**FINISHED MEASUREMENTS**` comment with
   ⚠️ alert lines for flagged measurements and 🔧 lines for auto-corrections.
5. **Optionally queues an Optitex job** so a Python agent on a Windows machine can drive
   Optitex 15 to generate the actual pattern file.

## The six workflows

### 1. Manual Calculator (`/` page)
User searches Asana for a task (or types measurements by hand), the engine runs **in the
browser**, results render with flags/ranges/corrections, then the user clicks "Post to Asana".

```
AsanaTaskLookup → GET /api/asana/search?q=        (typeahead search)
               → GET /api/asana/task/:id          (fetch + parse measurements)
Home.handleCalculate → calculateGarmentMeasurements()   ← runs client-side (Home.tsx:99)
ResultsDisplay "Post to Asana" → POST /api/asana/task/:id/measurements   (includes corrections)
ResultsDisplay "Send to Optitex" → POST /api/optitex/jobs                (priority 5)
```

Server then posts the Asana comment and upserts the `processed_orders` row to
`completed/postedToAsana=true` so the webhook won't double-post (routes.ts:223-267).

### 2. Batch Processing (`/batch` page)
User pastes task names or GIDs (newline/comma separated):

```
POST /api/asana/batch {taskIds}      → server resolves names→GIDs, fetches + parses each task
client filters: waist>10 && seat>10 && thigh>5 → otherwise "Skipped" card
client runs calculateGarmentMeasurements per task (browser-side)
"Post All"/"Post Selected"/individual → serial POST /api/asana/task/:id/measurements
```

Name resolution (routes.ts:114-152): a 15–16-digit number is used as a GID directly; otherwise
search with exact-match → substring-match → **first-result fallback** (can silently pick the
wrong task on typos).

### 3. Webhook Automation (the "hands-free" path)
This is the core production flow. One-time setup from the Automation page registers an Asana
webhook pointed at `https://<replit-domain>/api/asana/webhook`. Then:

```
Asana task created (e.g. from a Shopify order)
  → Asana POSTs events to /api/asana/webhook
  → HMAC-SHA256 signature verified against stored webhook secret (routes.ts:689-702)
  → 200 returned immediately; events processed async via setImmediate
  → processWebhookEvent (routes.ts:935-1183) per event:
      filters: task-type only, action === 'added' ONLY, dedupe (processed_orders row +
               hasFinishedMeasurements story scan), excluded project names,
               created_at >= 2024-11-01 and >= webhookRegisteredAt
      → fetchAsanaTask → parseAsanaMeasurements
      → normalize gender/style/fit (Skinny forces Trim)
      → sanity gate: waist>10 && seat>10 && thigh>5
      → calculateGarmentMeasurements()        ← runs server-side
      → postFinishedMeasurements comment to Asana
      → processed_orders row → status 'completed', postedToAsana=true
```

**Bold Metrics step (June 2026):** Virtual Tailor orders now arrive with the measurement fields
empty and the `VT *` input fields populated. Between parsing and the sanity gate, the pipeline
detects this (format `'unknown'` + VT inputs present), calls Bold Metrics server-side, awaits
the response, writes the six measurements into the Asana custom fields, re-fetches the task,
and continues normally. The billed result is persisted on the order row *before* the Asana
write so a retry never pays for a second call. Failed orders are no longer auto-reprocessed by
duplicate webhook events — the Automation page's Retry button (now functional) is the recovery
path. The webhook also now forwards corrections (🔧 lines). It still does **not** create
Optitex jobs.

### 4. Optitex Automation
The **only** producer of Optitex jobs is the Calculator's "Send to Optitex" button
(ResultsDisplay.tsx:113 → POST /api/optitex/jobs). Lifecycle:

```
pending  → (Python agent polls GET /api/optitex/jobs/poll every 5s, Bearer token)
         → agent PATCHes status 'processing'   (processingStartedAt stamped)
         → agent drives Optitex 15 via PyAutoGUI, saves
           C:\Optitex\Automation\Output\{CUSTOMER}_{STYLE_ABBREV} {FIT_ABBREV}_SKYNET.dxf
         → agent PATCHes 'completed' (+agentResponse) or 'failed' (+errorMessage)
JobsQueue page polls /api/optitex/jobs + /api/optitex/stats every 3s
```

⚠️ As shipped, every actual Optitex UI interaction in the agent is a commented-out TODO — the
agent reports success without creating a file. See doc 08.

### 5. Bold Metrics tab (`/bold-metrics` page)
In-Skynet replacement for the public Quick Tailor page (which gets removed from the storefront).
Staff load or search an Asana task (`acceptVTOnly` so VT-only orders load) or enter Virtual Tailor
inputs by hand (gender, age, height, weight, shoe size; **men require a preferred waist** +
optional inseam; women: bra size), fire Bold Metrics on demand (`POST /api/boldmetrics/calculate`,
rate-limited, server-side so the key stays secret), review the returned measurements with Skynet's
quarter-rounding preview, and post the six fields onto an Asana task
(`POST /api/asana/task/:id/boldmetrics`; returns **409 unless `overwrite=true`**). The returned
`waist` is the **average** of Bold Metrics' three waist circumferences. This is also the manual
recovery path when an automated VT order fails.

### 6. Full Workflow (`/full-workflow` page)
A single-screen, 3-column production dashboard that chains the two halves of the order flow:
**Bold Metrics (VT inputs) → Skynet inputs → Skynet results**. Both forms are always visible and
can auto-fire via a persisted **Auto-run on load** toggle (only a *fresh* VT order arms the billed
Bold Metrics call). It loads (or lets you manually select) an Asana task, decodes its `VT *` inputs
into the Bold Metrics form (`client/src/lib/vtInputs.ts`), maps gender/style/fit defaults
(`client/src/lib/asanaMapping.ts`), runs Bold Metrics + the spec engine, and posts back via two
post bars (a quick top bar and a full bottom bar). Payloads and 409-conflict messaging are built in
`client/src/lib/finishedMeasurements.ts`.

```
GET  /api/asana/task/:id          → measurements + vtInputs (returned together)
POST /api/boldmetrics/calculate   → measurements + skynetPreview + echoed gender (seeds Skynet form)
calculateGarmentMeasurements()    → results + patternReadiness  ← runs client-side
POST /api/asana/task/:id/boldmetrics   (409 unless overwrite=true)
POST /api/asana/task/:id/measurements  (includes corrections + patternReadiness)
```

## Pattern Readiness ("Pattern Guidance")

Every calculation now also produces a **pattern-readiness verdict**
(`shared/spec-engine/patternReadiness.ts` → `SpecOutput.patternReadiness`):

- `status`: `ready_for_pattern` | `needs_review` | `blocked`
- `summary`, `reasons[]` (blocking issues), `reviewPoints[]` (patternmaker attention),
  `adjustmentsApplied[]` (auto-corrections already applied)

`ResultsDisplay` renders it as a **Pattern Guidance** card and includes `patternReadiness` in the
post-to-Asana payload; the Pattern Queue dashboard (formerly Jobs Queue) shows it per job
(persisted in `optitex_jobs.pattern_readiness`). See doc 05 for the decision rules.

> **⚠️ Client-only.** Pattern readiness reaches Asana **only via the client posting paths**
> (`ResultsDisplay` / Full Workflow → `POST /api/asana/task/:id/measurements`). The **webhook
> automation path does not**: its `finishedMeasurements` object (in `server/routes.ts`,
> `processWebhookEvent`) omits `patternReadiness`, so an auto-processed order gets the
> measurements + 🔧 corrections comment but **no 🧭 Pattern Guidance section**. See doc 03.

## Who talks to whom

```
┌─────────────┐   webhooks + REST (PAT)    ┌──────────────────────────────┐
│    Asana    │◄──────────────────────────►│  Express server (Replit)     │
└─────────────┘                            │  - API + serves React SPA    │
                                           │  - spec engine (server-side) │
┌─────────────┐    same-origin fetch       │  - Drizzle ORM               │
│  React SPA  │◄──────────────────────────►│                              │
│  (browser)  │  spec engine also runs     └──────────┬───────────────────┘
│             │  client-side               WebSocket  │
└─────────────┘                            ┌──────────▼───────────────────┐
                                           │  Neon Postgres               │
┌───────────────────────┐  poll/PATCH      │  webhook_settings            │
│ Python agent          │  (Bearer token)  │  processed_orders            │
│ (Windows + Optitex 15)│◄────────────────►│  optitex_jobs                │
└───────────────────────┘   via Express    └──────────────────────────────┘
```

## Key invariants to preserve when making changes

- **The engine must stay shared.** `calculateGarmentMeasurements` runs in the browser (Home,
  BatchProcess) and on the server (webhook). Any change to calculation logic in
  `shared/spec-engine/` automatically affects all three paths — that's by design.
- **Dedupe depends on the comment footer.** `hasFinishedMeasurements` (asana.ts:433) keys on the
  literal footer text `*Auto-calculated by Garment Measurement Calculator*` plus keyword
  heuristics. Changing the comment format can break double-post protection.
- **Custom fields are matched by display name** (never GID) — e.g. `Waist Around`, `Waist Avg.`
  (trailing period required), `Hip Circum`. Renaming a field in Asana silently breaks parsing.
- **Middleware order is load-bearing.** `/api/asana/webhook` must get `express.raw` before the
  global `express.json` (server/index.ts:8-23) or HMAC verification breaks.
- **`main` deploys to production via Replit autoscale** — `npm run build` then `npm run start`.
  Type checking (`npm run check`) is NOT part of the build, so type errors do not block deploys.
