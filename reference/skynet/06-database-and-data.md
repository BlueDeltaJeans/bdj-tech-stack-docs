# 06 — Database & Reference Data

Two kinds of data:
- **Operational state** in Neon Postgres (webhook settings, processed-order audit, Optitex queue).
- **Measurement reference data** in version-controlled JSON (`shared/*.json`) bundled at build
  time — changing it requires a code deploy, not a DB write. Source Excel workbooks live in
  `knowledge_base/` and are only read by the offline stats generator.

## Postgres schema (`shared/schema.ts`)

Four tables. Schema sync is `npm run db:push` (`drizzle-kit push`) — **no migrations directory,
no versioned history**; pushes go straight to the database.

### `users` (schema.ts:6-10) — DEAD
`id` (uuid PK), `username` (unique), `password` (plaintext column). Auth scaffolding from the
Replit template; storage methods exist but **no route ever calls them**. No login system exists.

### `webhook_settings` (schema.ts:21-31) — singleton automation control
| Column | Notes |
|---|---|
| `enabled` boolean, default false | Master switch for webhook processing |
| `webhook_secret` text | Asana X-Hook-Secret stored here |
| `asana_project_ids` text[] | ⚠️ Write-only — stored via PUT but never consulted during event processing (filtering is the hardcoded exclusion list in routes.ts:1007) |
| `last_event_received_at`, `total_events_processed` | Counters (non-atomic updates) |
| `webhook_registered_at` | Catch-up event suppression cutoff |

Singleton by convention only (`select … limit 1`); nothing prevents multiple rows.

### `processed_orders` (schema.ts:45-60) — automation audit / idempotency
| Column | Notes |
|---|---|
| `task_id` text UNIQUE | Asana task gid — the natural key |
| `task_name` | Placeholder `Task ${taskId}` when created by the manual post path |
| `status` text | 'pending' \| 'processing' \| 'completed' \| 'failed' (TS-only typing; no pg enum/CHECK) |
| `body_measurements`, `garment_measurements` jsonb | Full calculation input/output snapshots |
| `gender`, `style`, `fit` | |
| `bold_metrics_response` jsonb | **Added 2026-06.** Persisted Bold Metrics result for VT orders, written the moment the (per-call-billed) API responds — **before** the Asana field write — so a retry recovers these values instead of re-billing. Shape `{ measurements, outlierMessage?, message? }` |
| `error_message`, `posted_to_asana`, `processed_at` | |

Rows accumulate forever — the only cleanup function (`cleanupOldProcessedOrders`) is uncalled
and buggy (deletes newest, not oldest; storage.ts:206).

### `optitex_jobs` (schema.ts:74-91) — cloud job queue
| Column | Notes |
|---|---|
| `status` | default 'pending'; pending → processing → completed/failed |
| `task_id`, `task_name` | nullable (manual-entry jobs have none) |
| `body_measurements`, `garment_measurements` jsonb NOT NULL | ⚠️ `garmentMeasurements.waist` is a **range string** like "37-37.5" — the agent expects a single number (see doc 08) |
| `gender`, `style`, `fit` NOT NULL | |
| `priority` int, default 5 | ⚠️ Stored but never used in any query — queue is `createdAt DESC` (LIFO) |
| `error_message`, `agent_response` jsonb | Agent posts results/screenshots-paths here |
| `pattern_readiness` jsonb | **Added 2026-06.** nullable; snapshot of the spec engine's `PatternReadiness` (status/summary/reasons/reviewPoints/adjustmentsApplied — see doc 05). Set on job create from the posted `patternReadiness` field |
| `processing_started_at`, `completed_at` | Stamped by storage on status transitions |

> ⚠️ **Schema sync:** both jsonb columns above (`processed_orders.bold_metrics_response`,
> `optitex_jobs.pattern_readiness`) were added on staging. Because there is no migrations directory,
> an existing database must run **`npm run db:push`** before code that writes them will work.

### Validation reality check
The drizzle-zod insert schemas (`insertUserSchema`, etc.) are used **only** to derive TypeScript
types via `z.infer` — there are zero `.parse`/`.safeParse` calls anywhere. Route handlers
validate request bodies manually or not at all; jsonb writes use `as any` casts.

Also defined in schema.ts (TypeScript only, not tables): `BodyMeasurements`,
`GarmentMeasurements`, the Gender/GarmentStyle/GarmentFit unions (single source of truth for the
whole app), and `WorkOrderData` — a 57-field interface that is referenced **nowhere** (the
work-order endpoint builds its response inline).

## Storage access (`server/storage.ts`)

Single `DatabaseStorage` implementation behind an `IStorage` interface, exported as the
`storage` singleton. See doc 03 for behavior notes (error swallowing on reads, key methods).

## Reference data

### `shared/outlier-stats.json` — outlier detection statistics

Structure: `{Women|Men: {<Style>: {sampleSize, confidence, ratios: {kneeToThigh, calfToKnee,
legOpenToThigh, seatToWaistDiff, fullRiseToSeat}}}}`; each ratio has
`{mean, stdDev, min, max, median, range95: [lo,hi], range99: [lo,hi]}` (range95/99 = mean ± 2σ/3σ).

Sample sizes (total ≈ 5,284 patterns — note replit.md's "10,000+" claim refers to the women's
thigh-table knowledge base, a different dataset):

| Gender | Style | n | confidence |
|---|---|---|---|
| Men | Straight | 2801 | excellent |
| Men | Boot | 684 | excellent |
| Men | Fashion Boot | 171 | good |
| Men | Skinny | 88 | moderate |
| Women | Straight | 655 | excellent |
| Women | Skinny | 425 | excellent |
| Women | Boot | 254 | excellent |
| Women | Flare | 164 | good |
| Women | Cigarette | 42 | insufficient — ⚠️ not a valid GarmentStyle; unreachable from typed code |

Missing combos (Men/Flare, Women/Fashion Boot) silently get **no** statistical checks.

⚠️ **The `fullRiseToSeat` blocks were added by hand**, not by the generator (Women 64.3 ± 3.5,
Men 67.5 ± 3.5, identical across styles within a gender). **Re-running the generator would
delete them** — see below.

### `scripts/generate-stats.ts` — the offline generator

Reads `knowledge_base/final_full_combined_wide_text.xlsx` (Women + Men sheets; columns `Style`,
`THIGH`, `KNEE`, `CALF`, `LEG OPEN`, `SEAT`, `WAIST`) and writes `shared/outlier-stats.json`.
Run manually (no package.json entry). Pipeline details:

- Style mapping (:81-90): STRT→Straight, SKNY→Skinny, BOOT, FLARE, FASH BOOT/FBOOT→Fashion Boot,
  CIG→Cigarette; unknown codes pass through uppercased (filtered later by the sample gate).
- Rows kept only when all six measurement columns are truthy; styles with < 30 samples skipped.
- Ratio validity filters: percentages 0 < v < 200; seat−waist 0 < diff < 30.
- Stats are slightly nonstandard: **population** stdDev (÷N), median = upper-middle element for
  even N, all values toFixed(2).
- Confidence tiers: ≥200 excellent, ≥100 good, ≥50 moderate, else insufficient.
- **Computes only four ratios** — no fullRiseToSeat. Re-running destroys the hand-added blocks
  that `outliers.ts` actively uses. If you regenerate, re-add them (or fix the script first).

### `shared/ease-rules.json` — historical ease medians (DEAD at runtime)

439 patterns of 2024-2025 data (men's style codes), generated 2025-10-02 by a script that was
**never committed** (likely from `knowledge_base/men_patterns_2024_2025.xlsx`, which no code
references). Structure: per style code (STRAIGHT n=385, FASHBOOT n=17, BOOT n=23, SKINNY n=1,
plus unreachable duplicates STRT n=11, FBOOT n=2) → median/q1/q3 ease for waist/seat/thigh/knee/
calf. Consumed only by `getEase` (spec-engine/ease.ts), which **no calculation path calls** —
live ease values are hardcoded in mens.ts/womens.ts. Keep in mind if anyone proposes
"data-driven ease": this file was the start of that idea.

### `knowledge_base/`

- `final_full_combined_wide_text.xlsx` (5.8 MB) — sole input to generate-stats.ts.
- `men_patterns_2024_2025.xlsx` (18 MB) — referenced by nothing; inferred source of
  ease-rules.json.

Neither is read at runtime.

## Where each kind of state lives

| State | Where |
|---|---|
| Webhook config + secret | Postgres `webhook_settings` |
| Order processing audit/idempotency | Postgres `processed_orders` |
| Optitex queue | Postgres `optitex_jobs` |
| Ease formulas | **Hardcoded** in mens.ts/womens.ts |
| Thigh ranges | **Hardcoded** tables in lookupTables.ts |
| Outlier statistics | `shared/outlier-stats.json` (bundled) |
| User auth | Nowhere (users table is dead scaffolding) |
| Calculated results | Asana comments + `processed_orders` jsonb snapshots |
| Bold Metrics billed result (retry-recovery cache) | Postgres `processed_orders.bold_metrics_response` |
| Pattern readiness for queued jobs | Postgres `optitex_jobs.pattern_readiness` |
