# BDJ-Tech-Stack-Docs-External

The **external, sanitized handoff bundle** of Blue Delta Jeans tech-stack documentation, given to
prospective agencies *before* an engagement is signed. Markdown only — no code, no build, no tests.
Remote: `BlueDeltaJeans/bdj-tech-stack-docs`. Status: a deliberately frozen point-in-time snapshot
(README says June 2026; `00-orientation.md` says "Last verified: 2026-06-24").

## 🔴 Everything in this repo is read by people outside the company

This is the one repo in the workspace where a mistake is a leak. Treat every file as **already
published to a stranger**. Never add — and actively strip if you are moving content in:

- credential **values**, tokens, keys, API secrets, connection strings, account logins
- **where** a secret is stored (secret-store names, vault paths, "it's in X's env")
- customer PII of any kind — real names, emails, addresses, measurements, order records
- internal-only decisions, staffing, vendor pricing, incident detail, or anything from
  `BlueDelta-Brain/raw-private/` or `wiki-private/` (never read those into this repo at all)

**Git history is part of the publication.** There is currently one commit (`e8c07a5`). If something
sensitive ever lands here, deleting it in a follow-up commit does **not** fix it — stop and escalate
to a human for a history rewrite / repo replacement.

### What *is* in-bounds (so you don't over-sanitize)

Private **repo names**, vendor/product names, architecture, and **environment-variable names** are
intentionally in plain text — the README states repos are named openly and shared once engaged.
The line is names vs. values-and-locations.

## Sync relationship with the internal hub

The sibling `../BDJ-Tech-Stack-Docs` (remote `BlueDeltaJeans/bdj-tech-stack-docs-internal`) is the
internal source. **Anything pulled from it must be sanitized before it lands here**, and the existing
export already encodes what that means:

- internal `06-integrations-and-credentials.md` → external `06-integrations.md`. The credentials/
  access matrix, the "ENV VAR → app/repo → **where it's set**" table, and the entire
  "Secrets & PII Handoff Rules" / "Where secrets actually live" sections are **dropped**; what
  survives is a names-only "Environment-variable names by code surface".
- internal `00-agency-onboarding.md` → external `00-orientation.md`.
- internal `08-dev-site-and-dashboard.md` and `09-jeanius-staging-build.md` have **no external
  counterpart**. The whole 2026 stack (`bdj-dev-site`, `blue-delta-dashboard`, `bdj-headless-staging`)
  appears **zero** times here. Do not "catch this bundle up" with them without an explicit decision —
  the omission is the design, not drift.
- Supabase: internal covers both the `bdj-measurements` and `blue-delta-dashboard` projects; external
  covers only `bdj-measurements`.

## Conventions to match when editing

- Example data is synthetic only — the established stand-in customer is **"Erin Test"**.
- Redact inside inlined exports as `<REDACTED — what it was>` (see `reference/zapier/zap-export json.md`).
- Unverified claims are flagged inline as `[CONFIRM]`.
- The bundle must stay **self-contained**: every link resolves to a file inside this folder. No links
  to Notion, private repos, or internal tools — inline the content into `reference/` or `appendix/`.

## Gotchas

- **`graphify-out/` is not protected by this repo.** It exists on disk, is tracked by zero files, and
  is **absent from `.gitignore`** — it survives only via a machine-global ignore
  (`~/.config/git/ignore`). On any other machine or in CI, `git add -A` would commit generated graph
  output into an outward-facing repo. Verify before any bulk `git add` here.
- **`README.md` is stale about its own contents.** It states that `00-orientation.md`,
  `06-integrations.md`, `reference/domain/domain-context.md`, and `appendix/notion-source-context.md`
  are "planned but not yet included in this build", and omits them from the document index — but all
  four exist and are substantial. Since outsiders read this file, fix the README rather than trusting it.
