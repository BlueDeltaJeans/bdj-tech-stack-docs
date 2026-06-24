# 02 — Architecture, Tech Stack & Deployment

## Tech stack at a glance

| Layer | Technology |
|---|---|
| Frontend | React 18 + TypeScript, Vite 5, wouter (routing), TanStack React Query 5, shadcn/ui + Radix primitives, Tailwind CSS 3, lucide-react icons |
| Backend | Express 4 (ESM, TypeScript run via tsx in dev, esbuild bundle in prod) |
| Shared logic | `shared/spec-engine/` — pure TypeScript calculation library imported by both client and server |
| Database | Neon serverless PostgreSQL (WebSocket transport via `ws`), Drizzle ORM, `drizzle-kit push` for schema sync (no migration files) |
| External | Asana REST API v1.0 (Personal Access Token), Optitex 15 via Python/PyAutoGUI agent |
| Hosting | Replit autoscale deployment; single process serves API + SPA on port 5000 → external :80 |

`package.json` name is `rest-express` — a Replit template leftover, not the project name.

## Repository layout

```
server/                  Express backend
  index.ts               Entry: middleware, error handler, dev/prod serving switch, listen
  routes.ts              ALL 26 HTTP routes + processWebhookEvent pipeline (incl. Bold Metrics)
  asana.ts               Asana REST client, measurement parsing, comment posting, webhooks
  asana.d.ts             DEAD: ambient types for the unused 'asana' npm package
  storage.ts             IStorage interface + DatabaseStorage (Drizzle) singleton
  db.ts                  Neon Pool + Drizzle instance (throws if DATABASE_URL unset)
  vite.ts                Dev: Vite middleware + HMR. Prod: static dist/public + SPA fallback
shared/                  Code shared between client and server
  schema.ts              Drizzle tables + ALL domain types (single source of truth)
  measurements.ts        Thin back-compat shim: calculateGarmentMeasurements → calculateSpecs
  spec-engine/           The calculation engine (14 files incl. patternReadiness.ts — see doc 05)
  ease-rules.json        Historical ease medians (439 patterns) — consumed only by dead code
  outlier-stats.json     Per-gender/style ratio statistics for outlier detection
client/                  React SPA (Vite root = client/)
  index.html             Shell (Google Fonts preloads, Replit dev-banner script)
  src/App.tsx            Router: / , /full-workflow , /batch , /bold-metrics , /automation , /jobs
  src/pages/             Home, FullWorkflow, BatchProcess, BoldMetrics, JobsQueue (Pattern Queue),
                         Automation, WorkOrder (orphaned), not-found
  src/components/        ResultsDisplay, MeasurementInput/Card, BoldMetricsForm, Flag/RangeIndicator,
                         AsanaTaskLookup/Selector, app-sidebar, ThemeToggle
  src/components/ui/     ~47 stock shadcn primitives (don't hand-edit)
  src/components/examples/  DEAD: design-phase demo wrappers, never imported
  src/lib/queryClient.ts Shared QueryClient + apiRequest + default queryFn
  src/lib/               + asanaMapping.ts, vtInputs.ts, finishedMeasurements.ts (2026-06 helpers)
scripts/
  generate-stats.ts      Offline generator: knowledge_base xlsx → outlier-stats.json
knowledge_base/          Historical pattern Excel workbooks (generator inputs, not runtime)
optitex_agent.py         Windows Python agent (job poller + PyAutoGUI driver)
test_connection.py       Agent pre-flight connectivity check
OPTITEX_AGENT_SETUP.md   Agent operator guide
optitex-python-agent.md  OLDER alternate agent design doc (diverges from shipped agent)
replit.md                Living project doc / changelog (partially stale)
design_guidelines.md     UI design spec (dark-first Linear/Notion aesthetic)
2025-style-migration-plan.md  Planned (NOT implemented) 2025 style expansion
attached_assets/         Replit Agent prompt history + design files — useful archaeology
```

## Path aliases

| Alias | Target | Defined in |
|---|---|---|
| `@/*` | `client/src/*` | vite.config.ts:20, tsconfig.json:19 |
| `@shared/*` | `shared/*` | vite.config.ts:21, tsconfig.json:20 |
| `@assets/*` | `attached_assets/*` | vite.config.ts:23 only — **no tsconfig entry**, invisible to tsc/IDE |

## Build & run

| Command | What it does |
|---|---|
| `npm run dev` | `NODE_ENV=development tsx server/index.ts` — TS server directly, Vite as middleware with HMR, single port |
| `npm run build` | `vite build` (client → `dist/public`) then `esbuild server/index.ts --platform=node --packages=external --bundle --format=esm --outdir=dist` (server → `dist/index.js`; node_modules stay external, so prod still needs `npm install`) |
| `npm run start` | `NODE_ENV=production node dist/index.js` — serves `dist/public` statically with SPA fallback |
| `npm run check` | `tsc` (noEmit). **Not wired into build/deploy** — run it manually. Excludes `**/*.test.ts`, so the spec-engine test file is never type-checked |
| `npm run db:push` | `drizzle-kit push` — pushes `shared/schema.ts` straight to the database. No `migrations/` directory exists; there is no versioned migration history |
| Engine tests | `npx tsx shared/spec-engine/calculateSpecs.test.ts` — plain script with throwing asserts, no test framework |

### Dev vs prod serving (server/index.ts:70-74, server/vite.ts)

- **Dev**: API routes registered first, then Vite middleware and an `index.html` transform
  catch-all (cache-busted per request with nanoid). HMR websocket piggybacks the same HTTP
  server. ⚠️ The custom Vite logger calls `process.exit(1)` on ANY Vite error (vite.ts:34-37) —
  a frontend compile error kills the whole backend in dev. `allowedHosts: true` disables host
  checking (needed for Replit).
- **Prod**: `serveStatic` resolves `dist/public` relative to the bundled server file via
  `import.meta.dirname` and throws at startup if missing. Unknown `/api/*` GETs fall through to
  `index.html` (HTML 200, not JSON 404) — this matters; see the broken Automation orders query
  in doc 09.

## Replit deployment (`.replit`)

- Modules: `nodejs-20`, `web`, `postgresql-16` (Replit-managed Postgres in the workspace).
- Run button: workflow runs `npm run dev`, waits for port 5000.
- Deployment: `deploymentTarget = "autoscale"`, build `npm run build`, run `npm run start`.
  Clicking **Publish** in Replit deploys and auto-commits `Published your App` to git
  (25 such commits — they are your deploy markers).
- Ports: localPort 5000 → externalPort 80. Only port 5000 is non-firewalled.
- `[agent]` integrations still list `asana:1.0.0` — inert leftover from the abandoned Replit
  OAuth connector (replaced by PAT in April 2026).

## Environment variables / secrets

| Var | Where | Purpose |
|---|---|---|
| `DATABASE_URL` | Replit secret | Neon Postgres connection. **Required** — both `server/db.ts:8` and `drizzle.config.ts:3` throw at load if unset |
| `ASANA_ACCESS_TOKEN` | Replit secret | Asana Personal Access Token; required for every Asana call (asana.ts:9-10). Sent as `Bearer` header. No OAuth, no expiry |
| `OPTITEX_AGENT_TOKEN` | Replit secret **and** agent's Windows `.env` | Shared bearer token for the agent's poll/PATCH endpoints. ⚠️ If unset on the server, any bearer value is accepted (routes.ts:792, 880) |
| `BOLDMETRICS_USER_KEY` | Replit secret | Bold Metrics API key (company-wide, one key). **Required** for the Virtual Tailor pipeline; never hardcode. Server throws a clear error per-call if missing |
| `BOLDMETRICS_CLIENT_ID` | Replit secret (optional) | Defaults to `bluedelta` |
| `BOLDMETRICS_BASE_URL` | Replit secret (optional) | Defaults to `https://api.boldmetrics.io/virtualtailor/get` |
| `REPLIT_DOMAINS` | Replit-injected | First comma-separated entry builds the webhook target URL (routes.ts:536-540) |
| `PORT` | `.replit` (=5000) | Listen port, default 5000 |
| `NODE_ENV` | package.json scripts | dev/prod switch |
| `REPL_ID` | Replit-injected | Gates the cartographer Vite plugin (Replit dev only) |
| `API_BASE` | Agent's Windows `.env` only | The deployed Replit app URL the agent polls |

`.gitignore` now ignores `.env` (fixed June 2026 — verified no `.env` or key was ever committed).

## Git history shape

~350 commits, Sept 2025 → June 2026. Heavy initial build Oct 2025 (219 commits), then sustained
measurement-logic tuning. Major arcs: April 2026 spec-engine refactor + Asana PAT migration;
ongoing thigh lookup-table dialing with pattern-maker-confirmed data points; June 2026
correction-display work in Asana comments. `replit.md`'s changelog is the human-readable history
but is stale in places (it lists 7 spec-engine modules; there are now 14 files).
