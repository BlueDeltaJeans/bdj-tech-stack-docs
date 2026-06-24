# 03 — Backend Server & API Reference

The entire backend is five files under `server/`. `routes.ts` (1,182 lines) holds every endpoint
plus the webhook automation pipeline.

> **📍 `file:line` citations may be stale.** `routes.ts` and `asana.ts` have **grown since this
> doc was written** — `processWebhookEvent`, the route handlers, and the `finishedMeasurements`
> literal have shifted by roughly **50–300 lines** from the numbers quoted below. Treat every
> `routes.ts:NNN` / `asana.ts:NNN` reference as approximate and **search by symbol name**
> (function/handler/route path) rather than jumping to the line number.

## server/index.ts — bootstrap & middleware order

Order matters (registration order = execution order):

1. **Webhook raw-body capture** (index.ts:8-20): `/api/asana/webhook` only gets
   `express.raw({type:'application/json'})`; the raw Buffer is stashed on `req.rawBody` for HMAC
   verification, then manually `JSON.parse`d into `req.body`. If parsing fails, the body silently
   stays a Buffer and downstream sees no events.
2. `express.json()` + `express.urlencoded()` for everything else (index.ts:23-24).
3. **API logger** (index.ts:26-54): monkey-patches `res.json`; logs
   `METHOD path status in Xms :: {body}` truncated to 80 chars for `/api/*` paths. Response
   bodies (measurement data) go to console logs.
4. `registerRoutes(app)` (routes.ts:10) — registers everything below, returns the `http.Server`.
5. **Error handler** (index.ts:59-65): responds `{message}` with status, then **`throw err;`** —
   re-throws after responding (classic Replit-template bug; produces unhandled-exception noise).
6. Dev → `setupVite`; prod → `serveStatic` (index.ts:70-74).
7. Listen on `PORT` (default 5000), host `0.0.0.0`, `reusePort: true`.

**There is no session middleware, no passport, no CORS, no helmet/CSP, and no login system.**
The `users` table and storage methods exist but nothing calls them. The only auth anywhere:
(a) HMAC webhook signature verification, (b) the optional Bearer token on the two Optitex agent
endpoints. Everything else is unauthenticated.

Every handler follows the same error pattern: try/catch → `console.error` → 500 with
`{error, message: error.message}` (raw internal error strings are leaked to clients).

## Endpoint reference

### Asana proxy (no auth)

| Method & path | Location | Behavior |
|---|---|---|
| `GET /api/asana/search?q=` | routes.ts:12 | Typeahead task search via `searchTasksByName`. Returns `{results: [{gid, name}]}` |
| `GET /api/asana/task/:taskId` | routes.ts:31 | Fetch + `parseAsanaMeasurements`. Returns `{taskId, taskName, measurements}` |
| `GET /api/asana/task/:taskId/details` | routes.ts:56 | Work-order fields + task stories (comments). Returns `{task, stories}` |
| `GET /api/asana/task/:taskId/debug` | routes.ts:81 | Dumps raw `custom_fields` — unauthenticated debug endpoint |
| `POST /api/asana/batch` | routes.ts:105 | Body `{taskIds: string[]}` (GIDs or names). Name resolution: 15-16-digit → GID passthrough; else search with exact → substring → **first-result fallback** (routes.ts:150-151). Rejects tasks missing Gender or Style. Returns `{successful, failed, total, successCount, failureCount}`. Contains leftover debug logging for GID '99178' / 'David Balducci' (routes.ts:160-174) |
| `POST /api/asana/task/:taskId/measurements` | routes.ts:223 | Body `{measurements: FinishedMeasurements}`. Posts the Asana comment (this is the only path where `corrections` reach Asana), then upserts `processed_orders` → completed/postedToAsana=true (placeholder name `Task ${taskId}` if new, routes.ts:250) |
| `GET /api/asana/work-order/:taskId` | routes.ts:270 | Maps ~37 custom fields by name into a flat work-order JSON. `customerName = task.name.split('|')[0]`. Special case `taskId === "manual-entry"` returns a blank template (whose shape differs slightly from the real branch — extra `address`/`belt` keys) |

### Bold Metrics (Virtual Tailor)

| Method & path | Auth | Behavior |
|---|---|---|
| `POST /api/boldmetrics/calculate` | none (rate-limited 10/min — interim until app auth) | Body `{gender:'m'\|'w', age, height (total inches), weight, shoeSize?, waistSize (required for men), inseam?, braSize?}`. Calls Bold Metrics server-side (the `user_key` never leaves the server) and returns `{measurements, skynetPreview, message, outlier, outlierMessage}`. **Each call is billed** |
| `POST /api/asana/task/:taskId/boldmetrics` | none | Body `{measurements:{waist,seat,thigh,knee,fullRise,inseam}, overwrite?}`. Writes the six Bold Metrics custom fields onto the task (by GID). Returns 409 if the task already has measurements and `overwrite` isn't set |

### Webhook settings & admin (no auth)

| Method & path | Location | Behavior |
|---|---|---|
| `GET /api/webhook/settings` | routes.ts:399 | Singleton settings row; lazily creates defaults `{enabled:false, asanaProjectIds:[], totalEventsProcessed:0}` |
| `PUT /api/webhook/settings` | routes.ts:422 | Accepts `{enabled?, asanaProjectIds?}` (manual typeof/Array checks, no zod) |
| `GET /api/webhook/orders` | routes.ts:459 | Query `status`/`limit`/`offset` → `processed_orders`, newest first. ⚠️ The Automation page actually requests `/api/webhook/orders/all` (path segment) which does NOT match this route — see doc 09 |
| `GET /api/webhook/stats` | routes.ts:485 | `{completed, failed, pending, total}` (no 'processing' bucket) |
| `GET /api/asana/projects` | routes.ts:499 | All projects across all workspaces (for the webhook-registration UI) |
| `GET /api/asana/webhooks` | routes.ts:513 | Lists registered Asana webhooks per workspace |
| `POST /api/asana/webhooks/register` | routes.ts:527 | Body `{projectGid}`. Target URL = `https://${REPLIT_DOMAINS[0]}/api/asana/webhook`. On success stamps `webhookRegisteredAt` (only if a settings row already exists, routes.ts:548-553) |
| `DELETE /api/asana/webhooks/:webhookGid` | routes.ts:577 | Deletes the Asana webhook |
| `POST /api/webhook/retry/:taskId` | routes.ts:708 | Reprocesses an order: mock `action:'added'` event through `processWebhookEvent` with `{isRetry:true}` (bypasses the catch-up date filter; failed orders are claimable again). Returns `{success, outcome: 'processed'\|'skipped'\|'failed', reason, order}`; restores dangling 'processing' rows to 'failed' when the retry itself skipped |

### Webhook receiver

`POST /api/asana/webhook` (routes.ts:645-735) — three phases:

1. **Handshake**: if header `X-Hook-Secret` present → persist secret into `webhook_settings`,
   echo it back, 200. ⚠️ Unauthenticated — anyone POSTing with that header overwrites the secret.
2. **Signature verification**: requires `X-Hook-Signature` (400 if missing); 403 if no stored
   secret. If `settings.enabled` is false → returns 200 **before** verifying. Signature =
   HMAC-SHA256 over `req.rawBody` keyed by the stored secret, compared with plain `!==`
   (not `crypto.timingSafeEqual`). On mismatch, both expected and received signatures are logged.
3. **Dispatch**: returns 200 immediately; events processed via `setImmediate` → per-event
   `processWebhookEvent`. Updates `totalEventsProcessed += events.length` (non-atomic
   read-modify-write) and `lastEventReceivedAt`. Failures are invisible to Asana (no retry signal).

### Optitex job queue

| Method & path | Auth | Location | Behavior |
|---|---|---|---|
| `POST /api/optitex/jobs` | **none** | routes.ts:739 | Requires bodyMeasurements, garmentMeasurements, gender, style, fit; optional taskId/taskName/priority (default 5) and `patternReadiness` (stored in `optitex_jobs.pattern_readiness`). Creates status `'pending'` |
| `GET /api/optitex/jobs/poll` | Bearer | routes.ts:781 | Returns pending jobs, `limit` default 1, ordered `createdAt DESC` (**LIFO; priority ignored; no atomic claim** — concurrent agents can double-process). Header required (401 if missing); ⚠️ token value only compared `if (agentToken && token !== agentToken)` — unset env var = any token accepted |
| `GET /api/optitex/jobs` | none | routes.ts:814 | List with status/limit/offset |
| `GET /api/optitex/stats` | none | routes.ts:835 | `{pending, processing, completed, failed, total}` |
| `GET /api/optitex/jobs/:id` | none | routes.ts:849 | Single job (404 if absent) |
| `PATCH /api/optitex/jobs/:id` | Bearer (same caveat) | routes.ts:869 | Body `status` ∈ pending/processing/completed/failed + optional errorMessage/agentResponse. Storage stamps `processingStartedAt`/`completedAt` on transitions |
| `DELETE /api/optitex/jobs/:id` | **none** | routes.ts:916 | Anyone can delete jobs |

## processWebhookEvent — the automation pipeline (routes.ts:1095+)

Returns `{outcome: 'processed'|'skipped'|'failed', reason?, dangling?}`. Step by step:

1. **Filter**: `resource_type === 'task'` and `action === 'added'` only, then the in-process
   `tasksInFlight` set drops same-process duplicate events.
2. **Dedupe**: skip if the `processed_orders` row is completed+postedToAsana; skip **failed**
   orders on webhook events (recovery is the Retry button — prevents duplicate `added` events
   from auto-re-billing Bold Metrics); skip if `hasFinishedMeasurements(taskId)` finds an
   existing results comment (⚠️ still fails OPEN on Asana API error).
3. **Atomic claim** (`storage.claimProcessedOrder`): `INSERT … ON CONFLICT DO UPDATE … WHERE
   status NOT IN ('processing','completed')` (a 'processing' claim older than 15 min is
   considered crashed and can be taken over). Durable in Postgres — guards across instances and
   restarts. Claim lost → skip.
4. Fetch the task; assert `task.gid === taskId`.
5. **Project exclusion**: skip if any membership project name contains `'in production'`,
   `'alt info requests'`, `'alt 2.0'`, or `'alteration'` (hardcoded; `settings.asanaProjectIds`
   is stored but **never consulted**).
6. **Date filters**: skip if `created_at < 2024-11-01` or `< settings.webhookRegisteredAt`
   (the latter is bypassed for retries). Missing `created_at` bypasses both with a warning.
7. `parseAsanaMeasurements(task)` (see doc 04).
8. **Bold Metrics step (VT orders)** — when format is `'unknown'` (no Bold Metrics fields, no
   manual tape fields on the just-fetched task):
   - **Recovery first**: if a previous run persisted a billed result on the order row
     (`processed_orders.bold_metrics_response`), reuse it — no second billed call.
   - Otherwise, if any `VT *` input field is populated: validate inputs (clear failure message
     if gender/age/height/weight are missing), `callBoldMetrics()` and **await**, persist the
     result on the order row **before** the Asana write, `writeBoldMetricsMeasurements()` (the
     six fields by GID), re-fetch + re-parse the task (now `bold_metrics` format), and carry
     Bold Metrics' own `outlier` message forward to the comment.
   - Neither stored result nor VT fields → falls through to the sanity gate as before.
9. **Normalize**: gender W/Women/Woman→Women, M/Men/Man→Men, default Men; style → enum, default
   Straight; fit: **Skinny always forces Trim**, Trim|Tight→Trim, Easy|Relaxed→Easy, default
   Regular.
10. Missing measurements default to 0; **sanity gate**: `waist > 10 && seat > 10 && thigh > 5`
    or the row is marked **failed** with a clear message (covers VT orders whose input fields
    Zapier hadn't populated yet — retryable once they exist).
11. `calculateGarmentMeasurements(...)` — destructures `{garmentMeasurements, corrections}`.
12. Build `FinishedMeasurements`: if input had an inseam, the garment **outseam** value is posted
    under the `inseam` label and outseam omitted (domain convention — same in both client
    paths). Waist keeps its range string (e.g. `"37-37.5"`). All outlier/flag fields,
    **corrections** (🔧 lines), and any `boldMetricsOutlierMessage` are forwarded.
    > **⚠️ Correction:** the webhook's `finishedMeasurements` object (routes.ts, search
    > `processWebhookEvent` → the `const finishedMeasurements = {…}` literal) **omits**
    > `patternReadiness`. The webhook therefore never posts the **🧭 Pattern Guidance** section —
    > `buildPatternGuidanceSection` in `asana.ts` only fires when `measurements.patternReadiness`
    > is supplied, which only the **client** posting paths (`POST /api/asana/task/:id/measurements`,
    > called from `ResultsDisplay` / Full Workflow) do. (The webhook doesn't compute pattern
    > readiness — `calculateGarmentMeasurements` here returns only `{garmentMeasurements,
    > corrections}`.)
13. `postFinishedMeasurements` → Asana comment → row updated to `completed` with stored
    measurements, gender/style/fit, `postedToAsana: true`.

## server/storage.ts — storage layer

`IStorage` interface + `DatabaseStorage` (Drizzle) singleton `storage`. Notable behaviors:

- **Read methods swallow errors** (catch → console → return `undefined`/`[]`/zeroes). A DB
  outage looks like "no data"; the webhook receiver would 403 "Webhook not configured".
  Write methods propagate.
- `getWebhookSettings()` = `select … limit 1` — singleton by convention, nothing prevents
  multiple rows.
- Processed orders keyed by Asana `task_id` (unique).
- `cleanupOldProcessedOrders(retentionDays)` (storage.ts:198-215): **dormant and inverted** —
  uses `gte(createdAt, cutoff)` so it would delete the *newest* completed orders; it has zero
  callers. The Automation UI's "cleaned up after 90 days" claim is not implemented.
- `updateOptitexJobStatus` stamps `processingStartedAt` / `completedAt` on transitions
  (storage.ts:272-278).
- `priority` on Optitex jobs is stored but never used in any query.

## server/db.ts

Neon serverless `Pool` over WebSockets (`neonConfig.webSocketConstructor = ws`). Throws at import
time if `DATABASE_URL` is unset. Exports `pool` and `db` (Drizzle bound to the full schema).
