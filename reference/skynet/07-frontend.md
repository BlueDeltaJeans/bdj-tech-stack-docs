# 07 — React Frontend (`client/`)

Vite + React 18 + TypeScript SPA. Routing via wouter, server state via TanStack React Query,
UI via shadcn/ui + Radix + Tailwind 3 (HSL CSS-variable tokens, dark-first; see
`design_guidelines.md`). **The calculation engine runs in the browser** — `Home` and
`BatchProcess` import `calculateGarmentMeasurements` from `@shared/measurements` directly; the
backend is used only for Asana I/O, the Optitex queue, and webhook admin.

## Shell & routing

- `main.tsx`: bare `createRoot(...).render(<App />)` — no StrictMode, no error boundary.
- `App.tsx` routes (wouter `<Switch>`):

| Route | Page |
|---|---|
| `/` | `Home` — the calculator |
| `/full-workflow` | `FullWorkflow` — load → Bold Metrics → Skynet → post, in one page |
| `/batch` | `BatchProcess` |
| `/bold-metrics` | `BoldMetrics` — Virtual Tailor → Bold Metrics (with task search/load) |
| `/automation` | `Automation` — webhook dashboard |
| `/jobs` | `JobsQueue` — Optitex queue monitor |
| (fallback) | `not-found` |

Reusable building blocks shared across the calculator, Bold Metrics, and Full Workflow pages:
`AsanaTaskLookup` (search + load; `acceptVTOnly` also loads VT-only tasks), `BoldMetricsForm`
(VT inputs + calculate + result preview), `MeasurementInput`, `ResultsDisplay` (`hidePostActions`
lets a parent own posting), and the lib helpers `asanaMapping` (string→enum + `clampStyleToGender`),
`vtInputs` (`decodeVTInputsToForm`), and `finishedMeasurements` (`buildFinishedMeasurementsPayload`,
`describeBoldMetricsConflict`).

⚠️ **There is no `/work-order/:taskId` route**, but `WorkOrder.tsx` exists and BatchProcess's
"Work Order" button navigates there (BatchProcess.tsx:311) — it lands on the 404 page. The page
and its API endpoint (`GET /api/asana/work-order/:taskId`) are otherwise functional; the route
registration is what's missing.

- `AppSidebar` (6 items): Calculator (`/`) · Full Workflow (`/full-workflow`) · Batch Process
  (`/batch`) · Bold Metrics (`/bold-metrics`) · **Pattern Queue** (`/jobs`, renamed from "Jobs
  Queue" — route unchanged) · Automation (`/automation`).
- `ThemeToggle`: localStorage `theme` + `.dark` class on `<html>`; no pre-hydration script, so
  first paint can flash the wrong theme.
- `index.css`: HSL tokens for light/`.dark`; semantic chart colors — **chart-2 = green
  (in range), chart-3 = amber (warning), chart-5 = red (error)**; custom `.hover-elevate`
  overlay system; `@media print` stylesheet for work orders.

## Query client (`lib/queryClient.ts`) — read this before adding queries

- Defaults: `staleTime: Infinity`, `retry: false`, no window-focus refetch — data fetched once
  never refreshes unless a query sets `refetchInterval` or something calls `invalidateQueries`.
- The default `queryFn` builds the URL by **joining queryKey segments with `/`**.
  `['/api/webhook/orders', statusFilter]` ⇒ `GET /api/webhook/orders/all` — a path, not
  `?status=`. ⚠️ This is why the Automation orders list is broken (doc 09 #3). Adding a param to
  a queryKey silently changes the URL.
- `apiRequest(method, url, data?)` is the mutation helper (throws on non-OK).

## Pages

### Home (`/`) — Calculator
1. `AsanaTaskLookup` loads a task's parsed measurements into the form (rejects
   `format:'unknown'`).
2. `handleAsanaTaskFetched` (Home.tsx:31) maps raw gender/style/fit strings to enums (same
   mapping as the server; Skinny→Trim enforcement here relies on MeasurementInput's effect).
3. `handleCalculate` (Home.tsx:91): 300ms artificial delay, then **client-side**
   `calculateGarmentMeasurements`. Errors only `console.error`'d — no toast.
4. Results render in `ResultsDisplay`; `originallyHadInseam` tracks whether to post the garment
   outseam under the "Inseam" label.

### BatchProcess (`/batch`)
- Textarea → split on newlines/commas → `POST /api/asana/batch`.
- Client-side validity gate `waist>10 && seat>10 && thigh>5` → "Skipped Tasks" card.
- Per-task client-side calculation; fit mapping adds `Tailored→Regular`; Skinny forces Trim.
- `finalInseam = roundQuarter(inseam − 0.5 + (garmentOutseam − (inseam − 0.5)))` — algebraically
  just `roundQuarter(garmentOutseam)`; both this and ResultsDisplay post the garment **outseam**
  under the `inseam` field when the input had an inseam (domain convention, consistent with the
  webhook path).
- Post individual / selected / all → serial `POST /api/asana/task/:id/measurements` (includes
  `corrections`). ⚠️ The serial loop aborts on first failure, leaving later rows stuck with
  `posting: true` spinners.
- Flag icons: waist (flags or outlier), thigh (`thighFlag`), knee/legOpening (outlier-driven),
  calf (`calfFlag` **or** calf outlier), frontRise (`frontRiseFlag` **or** fullRise outlier);
  red when the driving outlier severity is `error`, else amber.

### Pattern Queue (`/jobs`) — formerly JobsQueue (`JobsQueue.tsx`)
Reworked (2026-06) into a "Pattern Production Queue" dashboard (sidebar label "Pattern Queue").
- Still polls `GET /api/optitex/jobs` + `GET /api/optitex/stats` every **3s** (toggleable) and
  supports delete.
- Each job now carries a `patternReadiness` field (`deriveLegacyPatternReadiness` synthesizes one
  for jobs predating the column). Cards show a production-status badge (Ready to Draft / Drafting
  Pattern / Draft Created / Blocked) **plus** a readiness badge (Ready for Pattern / Needs Review /
  Blocked), expandable **Pattern Guidance** and **View Specs** sections, and disabled workflow
  buttons (Create Pattern / Approve Pattern / Return for Review).
- A **Dev mode** checkbox gates the raw `agentResponse` panel, which is now parsed inside a
  try/catch — non-JSON no longer crashes the page (the old warning is resolved).

### Automation (`/automation`)
- `GET /api/webhook/settings` (fetched once, no polling), `GET /api/webhook/stats` (30s poll),
  orders query (30s poll, only when enabled) — now uses an explicit queryFn hitting
  `/api/webhook/orders?status=…`, so the Processed/Failed order lists render (fixed; was doc 09 #3).
- Toggle → `PUT /api/webhook/settings {enabled}`. `asanaProjectIds` is not editable from the UI.
- Retry button → `POST /api/webhook/retry/:taskId` — now actually reprocesses and reports the
  outcome (fixed; was doc 09 #2).
- Cosmetic copy: monitored projects "Auto | Pant Pipeline" / "Order Pipeline" are hardcoded
  display text; "~3,000 orders per month" and "cleaned up after 90 days" are prose, not enforced.

### WorkOrder (orphaned)
Printable work-order form: seeds client/dates from `GET /api/asana/work-order/:taskId` and
finished specs from the `?measurements=` URL param (`JSON.parse` without try/catch — malformed
param white-screens). Print via `window.print()` + the print stylesheet. Hardcoded raw-spec
placeholders (Knee "11", Calf "16", "X"). Unreachable until a route is added to App.tsx.

### FullWorkflow (`/full-workflow`) — added 2026-06
A wide (`min-w-[2000px]`) single-page console that runs the whole pipeline for one order:
load → Bold Metrics → Skynet → post.

- **Layout:** a top row (slim `AsanaTaskLookup` + an always-visible "quick" post card) over a
  3-column form grid — **Bold Metrics (`BoldMetricsForm layout="stacked"`) · Skynet inputs
  (`MeasurementInput`) · Skynet results (`ResultsDisplay hidePostActions`)** — over a full post bar.
- **Auto-run toggle** (`switch-auto-run`) is persisted to `localStorage` (`fullWorkflow.autoRun`).
  When on, loading a task runs Bold Metrics + Skynet automatically. Auto-run of the *billed* Bold
  Metrics call is additionally gated by `bmAutoArm`, true **only for a fresh VT order** (no
  measurements yet) — re-opening an already-measured VT order never re-bills.
- **Post target:** `activePostTaskId = loaded task || manually-selected task`. When no task is
  loaded, an `AsanaTaskSelector` in the full post bar picks a target.
- **Dual post bars:** the same three buttons (Post Body / Post Finished / Post Both) render in both
  the always-visible quick bar (top) and the full bar (bottom). The quick bar's buttons carry
  `tabIndex={-1}` so keyboard tabbing flows through the forms and lands on the bottom bar after
  Calculate (quick-bar `data-testid`s get a `-quick` suffix).
- **Posting:** body measurements → `POST /api/asana/task/:id/boldmetrics` (a 409 surfaces an
  in-page "Overwrite" prompt via `describeBoldMetricsConflict`); finished measurements →
  `POST /api/asana/task/:id/measurements` (payload from `buildFinishedMeasurementsPayload`,
  including `corrections` and `patternReadiness`). "Post Both" posts BM then Skynet sequentially,
  skipping a step that already succeeded.
- The Skynet `MeasurementInput` is keyed by `skynetFormKey` (bumped on every load) so it fully
  remounts and never keeps a prior task's body measurements on screen.

### BoldMetrics (`/bold-metrics`) — gained task search/load
Standalone Virtual-Tailor → Bold Metrics tool. Now includes **task search/load** (`AsanaTaskLookup`
with `acceptVTOnly`): loading a VT order prefills the form from `decodeVTInputsToForm(vtInputs)`.
Posts the six Bold Metrics fields to `POST /api/asana/task/:id/boldmetrics`; post target is a loaded
task or a manually selected one, with the same 409 overwrite prompt as Full Workflow.

## New / changed components & helpers (2026-06)

- **`BoldMetricsForm`** — the VT input form + raw/Skynet-preview result table. Props: `initialInputs`
  (prefill from a loaded task, full-resets on change), `onResult`, `layout` (`split` default /
  `stacked` for the dashboard), `autoRunOnLoad` (fires a **billed** calc on a fresh valid prefill).
  Men require a Preferred Waist; `buildBmRequest` is the shared validator/request-shaper used by both
  the manual button and auto-run (so auto-run uses the loaded values, not stale state).
- **`AsanaTaskLookup`** — new `acceptVTOnly` (also load VT-only tasks that parse as
  `format:'unknown'` but have `vtInputs.hasVTFields`) and `statusNote` (content under the search box,
  e.g. the "Loaded: …" line). Calculator leaves `acceptVTOnly` off.
- **`ResultsDisplay`** — new `patternReadiness` (renders the **Pattern Guidance** card: status badge
  + Blocking Issues / Review Points / Adjustments Applied lists) and `hidePostActions` (hide the
  Post/Optitex/task-selector controls so a parent like Full Workflow owns posting). Both it and Full
  Workflow build the post payload via `buildFinishedMeasurementsPayload`.
- **lib helpers** — `asanaMapping.ts` (`mapGender`/`mapStyle`/`mapFit`, `mapAsanaTaskToDefaults`,
  `STYLES_BY_GENDER` — now the single source for the style dropdown, `clampStyleToGender`),
  `vtInputs.ts` (`decodeVTInputsToForm`: total-inches→feet+inches, `"34C"`→band+cup),
  `finishedMeasurements.ts` (`buildFinishedMeasurementsPayload`, `describeBoldMetricsConflict`).

## Key components

| Component | Role |
|---|---|
| `ResultsDisplay` | Grid of MeasurementCards; "N warnings" header counts only `hasFlag` items (outlier-only issues are excluded from the count — header can say "All in range" while red outlier boxes show). Corrections card lists reason + original/corrected values + "Review required" tag. Post-to-Asana mutation (includes corrections). Send-to-Optitex mutation (⚠️ always sends the original prop `asanaTaskId`, not a manually selected task — manual-entry jobs get null task linkage) |
| `MeasurementInput` | Controlled form. Defaults: waist 34, seat 40, thigh 22, knee 16, calf 15, fullRise 27, outseam 42. Skinny forces+disables fit=Trim. Style lists: Women [Skinny, Straight, Boot, Flare], Men [Straight, Boot, Fashion Boot, Skinny]. ⚠️ 0 is coerced to undefined for inseam/suggestedFrontRise/suggestedLegOpening — an intentional 0 can't be entered |
| `MeasurementCard` | Body vs garment values, `parseRange("min-max")`, separate flag and outlier message boxes (independent channels). Body row hidden when bodyValue is 0/undefined |
| `FlagIndicator` / `RangeIndicator` | Green check "In Range" vs amber/red triangle; progress-bar position within min–max (clamped) |
| `AsanaTaskLookup` / `AsanaTaskSelector` | Debounced search comboboxes (near-duplicates; Selector returns gid+name only, used inside ResultsDisplay for manual-entry posting) |

`components/ui/*` is stock shadcn (~47 files). `components/examples/*` is dead design-phase code.
`hooks/use-toast.ts` is the toast state machine every page uses; `hooks/use-mobile.tsx` is the
responsive breakpoint hook.

## Duplication to be aware of

Gender/style/fit string→enum mapping is copy-pasted in three places (Home.tsx, BatchProcess.tsx,
server routes.ts webhook pipeline) with slightly different completeness; Skinny→Trim is enforced
in MeasurementInput, BatchProcess, asana.ts, and routes.ts; `roundQuarter` is redefined locally
in BatchProcess; AsanaTaskSelector duplicates AsanaTaskLookup's search logic. Consolidating these
into `shared/` would be a high-value cleanup before adding new styles.
