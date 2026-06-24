# Skynet — Technical Documentation

**Skynet** is Blue Delta Jeans' internal garment-measurement checker. It converts customer body
measurements into finished garment specifications for the pattern-making team, validates every
measurement against standard ranges built from historical pattern data, flags anything out of
range, and posts the results back to Asana. It was originally built in Replit (where it is
deployed and runs in production today).

This documentation was generated from a full audit of the codebase (June 2026, `staging` branch)
and is intended as the foundation for planning new features. Every claim cites `file:line` so you
can verify against source.

## Reading order

| Doc | Contents |
|---|---|
| [01-overview.md](01-overview.md) | What Skynet does, the six workflows, Pattern Readiness, and the end-to-end data flows |
| [02-architecture-and-stack.md](02-architecture-and-stack.md) | Tech stack, repo layout, build pipeline, Replit deployment, environment variables |
| [03-backend-api.md](03-backend-api.md) | Express server: every HTTP endpoint, middleware, the webhook processing pipeline |
| [04-asana-integration.md](04-asana-integration.md) | Asana REST client, custom-field measurement parsing, webhook lifecycle, comment posting |
| [05-spec-engine.md](05-spec-engine.md) | The calculation engine: ease rules, style formulas, corrections, outlier detection |
| [06-database-and-data.md](06-database-and-data.md) | Postgres schema, storage layer, and the statistical reference data (JSON + Excel) |
| [07-frontend.md](07-frontend.md) | React app: pages, components, state management, what calls which API |
| [08-optitex-agent.md](08-optitex-agent.md) | The Windows Python agent that polls the job queue and drives Optitex 15 |
| [09-known-issues-and-tech-debt.md](09-known-issues-and-tech-debt.md) | **Read before planning changes** — confirmed bugs, security gaps, dead code, fragile spots |

## One-paragraph mental model

A single Express server (port 5000) serves both the JSON API and the React SPA. The calculation
engine lives in `shared/spec-engine/` and runs **both in the browser** (manual calculator, batch
page) **and on the server** (webhook automation) — the same pure TypeScript module is bundled into
both. Asana is the system of record: a webhook fires when a new order task is created, the server
parses body measurements out of the task's custom fields, runs the engine, and posts a
"FINISHED MEASUREMENTS" comment back with ⚠️ warnings. Operational state (webhook settings,
processed-order audit trail, Optitex job queue) lives in a Neon Postgres database via Drizzle ORM.
A separate Python agent on a Windows machine polls the job queue to drive Optitex 15 pattern
creation via PyAutoGUI (currently a scaffold — see doc 08).

## Branch policy

`main` is **live production** (Replit deploys from it; "Published your App" commits are deploy
markers). All development happens on `staging` or feature branches off it. Never push directly
to `main`.
