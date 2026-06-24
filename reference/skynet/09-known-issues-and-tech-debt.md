# 09 — Known Issues, Security Gaps & Tech Debt

Everything here was verified against source during the June 2026 audit. Ordered by how much it
should influence feature planning.

> **Status update (June 2026, staging):** items 1–3 and 13 were FIXED as part of the Bold Metrics
> migration; item 5 was clarified as not-a-product (men's Flare isn't sold — men have Fashion
> Boot; current styles are Men: Straight/Boot/Skinny/Fashion Boot, Women: Straight/Boot/Skinny/
> Flare); item 7 (Optitex agent) is acknowledged scaffolding for a future project; item 9 (no
> auth) is a deliberate deferral — auth will be added after the new features, before staging
> goes live. Struck-through line references below may have shifted.

## Confirmed functional bugs

1. ~~**Webhook auto-processing silently drops corrections.**~~ **FIXED (staging):** the webhook
   path now destructures `corrections` and forwards them, so 🔧 Correction lines appear on
   auto-processed orders just like manual/batch posts.

2. ~~**The webhook retry endpoint is a no-op.**~~ **FIXED (staging):** the mock event now uses
   `action:'added'`, `processWebhookEvent(event, settings, {isRetry:true})` bypasses the
   webhook-registration catch-up date filter for retries, returns a structured
   `{outcome, reason}`, and the endpoint restores dangling 'processing' rows to 'failed' with
   the skip reason. The Automation page toast now reports the real outcome.

3. ~~**The Automation page's orders list can never render.**~~ **FIXED (staging):** the orders
   query now uses an explicit queryFn hitting `/api/webhook/orders?status=...` (and the server
   treats `status=all` as no filter).

4. **The Work Order feature is unreachable.** `WorkOrder.tsx` and its API endpoint work, but
   App.tsx has no `/work-order/:taskId` route — BatchProcess's "Work Order" button lands on the
   404 page (BatchProcess.tsx:311, App.tsx:15-26). *(Still open — not in scope for the Bold
   Metrics migration.)*

5. **Men's Flare is a silent hybrid.** mens.ts has no Flare branch: Men+Flare gets Fashion Boot
   knee/calf (mens.ts:95-105) and Straight legOpening/outseam (mens.ts:118-123). *(Clarified:
   men's Flare is not a product — the path is theoretically reachable from bad data but doesn't
   occur in practice. Men have Fashion Boot instead of Flare.)*

6. **`cleanupOldProcessedOrders` is inverted and dormant.** It deletes completed orders
   `createdAt >= cutoff` — the *newest* ones (storage.ts:206) — and has zero callers. The
   Automation UI's "cleaned up after 90 days" is not implemented; `processed_orders` and
   `optitex_jobs` grow forever.

7. **Optitex agent is a scaffold that reports false success** — all four UI-automation steps are
   TODOs; jobs get marked `completed` with paths to files that were never created (doc 08).

8. **Optitex waist mismatch.** `garmentMeasurements.waist` is a range string ("37-37.5"); the
   agent types it verbatim into a numeric Optitex field while its comments expect "the larger
   value" (optitex_agent.py:285-289).

## Security gaps

These are tolerable only because the app is an internal tool, but several are cheap to fix and
should be addressed before any feature increases exposure:

9. **No authentication on the API.** No sessions, no login, no CORS, no helmet. Anyone who can
   reach the Replit URL can: read/search Asana tasks through the server's PAT, **post comments to
   arbitrary Asana tasks**, toggle automation, register/delete Asana webhooks, create/delete
   Optitex jobs, and hit the raw-custom-fields debug endpoint
   (`GET /api/asana/task/:id/debug`). The `users` table + passport deps are unused scaffolding.
   *(Decision: deliberately deferred — auth will be added after the Bold Metrics features, before
   staging is pushed live. Interim mitigation: the billed `POST /api/boldmetrics/calculate`
   endpoint is rate-limited to 10 calls/minute.)*

10. **Unauthenticated webhook-secret overwrite.** Any POST to `/api/asana/webhook` with an
    `X-Hook-Secret` header replaces the stored secret (routes.ts:648-668) — an attacker can break
    verification or install their own secret and forge "validly signed" events.

11. **Optitex bearer-token fail-open.** `if (agentToken && token !== agentToken)`
    (routes.ts:792, 880): if the `OPTITEX_AGENT_TOKEN` env var is unset, any bearer value passes.

12. **HMAC compared with `!==`** instead of `crypto.timingSafeEqual`; on mismatch the expected
    signature is printed to logs; and when automation is disabled the endpoint 200s *before*
    verifying (routes.ts:684-702).

13. ~~**`.env` is not gitignored** while `.env.example` instructs copying to `.env`.~~
    **FIXED (staging):** `.env` is now in `.gitignore`; only `.env.example` is tracked.

14. **Error messages leak internals** — every handler returns `error.message` to the client;
    the API logger writes response bodies (customer measurement data) to console logs.

## Reliability / correctness risks

15. **Dedupe fails open.** `hasFinishedMeasurements` returns `false` on any Asana API error
    (asana.ts:458-461) — a transient error during webhook dedupe can double-post.
16. **Batch name resolution can pick the wrong task** (first-search-result fallback,
    routes.ts:150-151).
17. **No atomic job claim + LIFO queue.** Poll doesn't flip jobs to processing; two agents can
    double-process; `priority` is stored but ignored; newest jobs are served first so old jobs
    starve (storage.ts:239).
18. **Storage reads swallow errors** → DB outage looks like "no data" (and the webhook receiver
    would 403 "Webhook not configured").
19. **Error handler re-throws after responding** (index.ts:64); dev Vite logger
    `process.exit(1)`s the whole backend on any frontend compile error (vite.ts:34-37).
20. **`settings.asanaProjectIds` is write-only** — actual project filtering is the hardcoded
    exclusion list `['in production','alt info requests','alt 2.0','alteration']`
    (routes.ts:1007-1012). The Automation UI's project names are cosmetic.
21. **Regenerating outlier stats destroys data** — `generate-stats.ts` emits only 4 ratios;
    the hand-added `fullRiseToSeat` blocks the engine uses would be deleted (doc 06).
22. **`npm run check` is not in the build** and tsconfig excludes `*.test.ts`; the engine test
    script runs in no CI. Type errors and regressions cannot block a deploy.
23. **No DB migrations** — `drizzle-kit push` straight to prod, no versioned history.

## Dead code & template residue (safe-to-delete candidates)

| Item | Evidence |
|---|---|
| `users` table + storage methods + passport/express-session/connect-pg-simple/memorystore deps | zero callers |
| `asana` npm package + `server/asana.d.ts` + `@replit/connectors-sdk` | nothing imports them since the April 2026 PAT rewrite |
| `getEase` + `shared/ease-rules.json` | exported, never called; live ease is hardcoded |
| `generateWarnings` / `SpecWarning[]` output | computed every calculation, consumed by nothing (shim drops it) — *candidate for revival rather than deletion* |
| `client/src/pages/WorkOrder.tsx` | orphaned (or re-add the route — product decision) |
| `client/src/components/examples/*` | design-phase demos, never imported |
| `WorkOrderData` interface (schema.ts:165-238, 57 fields) | referenced nowhere |
| Debug block for GID '99178' / 'David Balducci' (routes.ts:160-174); `GET /api/asana/task/:id/debug` | leftover troubleshooting |
| `@tailwindcss/vite` (v4 plugin) in devDeps | project uses Tailwind v3 via PostCSS |
| `randomUUID` import in storage.ts:17 | unused |
| Root-level debug screenshots (results.png, jobs_after_reload.png, work-order-full.png) | committed binaries |
| Dead `confidence === 'insufficient'` check (outliers.ts:34) | confidence lives at style level, always undefined per-ratio |

## Duplication — consolidated (2026-06) and outstanding

**Consolidated by the Bold Metrics migration:**
- **Gender/style/fit mapping** — `Home.tsx`'s inline copy moved to `client/src/lib/asanaMapping.ts`
  (`mapGender`/`mapStyle`/`mapFit`/`mapAsanaTaskToDefaults`), now used by Home + Full Workflow.
  *Still inline:* `BatchProcess.tsx` and the `routes.ts` webhook pipeline (server-side).
- **Per-gender style lists** — now a single `STYLES_BY_GENDER` in `asanaMapping.ts`, imported by
  `MeasurementInput`'s dropdown and `clampStyleToGender`. No longer hardcoded in MeasurementInput.
- **Finished-measurements payload / inseam-under-"Inseam" convention** — `ResultsDisplay` and Full
  Workflow now share `buildFinishedMeasurementsPayload` (`lib/finishedMeasurements.ts`). The
  `routes.ts` webhook copy remains an independent third implementation.

**Still outstanding:**
- Skinny→Trim enforced in **five** places (MeasurementInput, BatchProcess, asana.ts, routes.ts, and
  `FullWorkflow.tsx`'s `runSkynet`).
- `roundQuarter` defined in three places (utils.ts, asana.ts, BatchProcess.tsx).
- `AsanaTaskSelector` ≈ `AsanaTaskLookup`.

## Stale documentation to be aware of

- `replit.md` lists 7 spec-engine modules; the directory has **14 files** (matching doc 02 and the
  actual `shared/spec-engine/` listing). It also claims "10,000+
  historical patterns" for outlier detection (actual outlier-stats total ≈ 5,284; the 10,496
  figure belongs to the women's thigh table) and describes a "PostgreSQL session store for
  authentication" that doesn't exist.
- `OPTITEX_AGENT_SETUP.md` line references to the TODO sections are stale.
- `2025-style-migration-plan.md` is explicitly unimplemented ("DO NOT implement yet") and
  internally inconsistent (says 12 women's styles / 8 new, enumerates 11 / 7).
- `Automation.tsx` UI copy (project names, 3,000 orders/month, 90-day retention) is cosmetic.

## Notable behavioral quirks (intentional-looking, verify with the team)

- **Garment outseam is posted under the "Inseam" label** whenever the input had an inseam —
  consistent across all three posting paths, so probably a domain convention; confirm with the
  pattern team before "fixing".
- Bold Metrics values are quarter-rounded on import; manual tape values are not.
- Hardcoded webhook date floor `2024-11-01` (routes.ts:1030); tasks with no `created_at` bypass
  all date filters.
- Gender dispatch defaults to Men's path for any unrecognized gender string; the agent's gender
  check maps anything ≠ "Men" to Womens patterns — two different defaults for bad data.
- An explicit 0 cannot be entered for inseam/suggestedFrontRise/suggestedLegOpening (truthy
  checks + 0→undefined coercion).
- `flagCount` in ResultsDisplay ignores outlier-only issues — header can say "All measurements
  in range" above visible red outlier boxes.
