# 08 — Optitex Python Agent (Windows)

A Python agent meant to run on a Windows PC with Optitex 15. It polls the cloud job queue every
5 seconds and drives Optitex via PyAutoGUI to create pattern files.

> ⚠️ **Critical status note:** as committed, every actual Optitex UI interaction — window focus,
> file open (Ctrl+O), measurement-tab clicking, and Save As — is a **commented-out TODO**
> (optitex_agent.py:247-251, 261-266, 209-216, 299-304). What actually executes is
> `keyboard.write(value)` + Enter into *whatever window has focus*, after which the agent returns
> success and the job is marked `completed` in the cloud with a filename for a file that was
> never created. The setup guide's pre-flight checklist confirms this ("Optitex UI automation
> implemented (TODO sections)", OPTITEX_AGENT_SETUP.md:244). If the production Windows machine
> runs a completed version of this script, that version is **not in this repo**.

## Files

| File | Role |
|---|---|
| `optitex_agent.py` (389 lines) | The agent: poll loop, status reporting, automation scaffold |
| `test_connection.py` | One-shot pre-flight check: validates `.env`, hits the poll endpoint, prints 200/401/403 diagnostics |
| `OPTITEX_AGENT_SETUP.md` | Operator guide (its "Line ~X" references to the TODOs are stale) |
| `optitex-python-agent.md` | **Older divergent design doc** with an alternate unauthenticated skeleton (different measurement order, double-encoded agentResponse, simulated success). Don't copy from it |
| `.env.example` | Template: `API_BASE` (Replit app URL) + `OPTITEX_AGENT_TOKEN` |

## Configuration

- `.env` via python-dotenv: `API_BASE`, `OPTITEX_AGENT_TOKEN` (must match the Replit secret).
- Hardcoded in source (lines 31-32): `BASE_PATTERN_FOLDER = C:\Optitex\Automation\BasePatterns`,
  `OUTPUT_FOLDER = C:\Optitex\Automation\Output`.
- `POLL_INTERVAL = 5` s; HTTP timeouts 10 s; `pyautogui.PAUSE = 0.5`; `FAILSAFE = True`
  (mouse to screen corner aborts).
- Dependencies: `requests python-dotenv pyautogui keyboard pillow`.

## Job loop (`OptitexAgent.run`, lines 338-380)

```
forever:
  jobs = GET {API_BASE}/api/optitex/jobs/poll        (Bearer token; server default limit=1)
  for job in jobs:
      PATCH /api/optitex/jobs/{id} {status: 'processing'}
      result = automate_optitex(job)
      PATCH {status: 'completed', agentResponse: {...}}        on success
      PATCH {status: 'failed', errorMessage, agentResponse}    on failure
  sleep 5s
```

- Status-PATCH failures are print-and-continue — **no retry**; a job can finish locally while
  the cloud row stays `processing` forever. There is no stuck-job reaper anywhere.
- No atomic claim: poll returns pending rows without transitioning them, so two agents can
  both receive the same job. (The old design doc's claim that agents "automatically coordinate"
  is false.)
- Queue order is `createdAt DESC` (newest first / LIFO) and `priority` is ignored server-side.

## automate_optitex (lines 226-336)

1. **Focus Optitex window** — TODO stub.
2. **Pick base pattern**: `find_closest_base_pattern(gender, seat)` globs
   `{Mens|Womens}Base_*.dxf` and picks the file whose filename seat number is closest to
   `garmentMeasurements.seat`. Gender prefix: `"Mens"` only when gender is exactly `"Men"`;
   anything else → `"Womens"`. Missing seat defaults to 0 → smallest pattern. Filename parsing
   assumes exactly `{Prefix}Base_{number}.dxf`. Opening the file is a TODO stub.
3. **Enter measurements** in fixed order: knee → calf → legOpening → seat → waist, each
   `measurements.get(key, 0)` (missing values silently type 0), each typed + Enter + 1s wait.
   ⚠️ Waist: the engine produces a **range string** ("37-37.5") but the agent types the value
   verbatim, with a comment assuming "backend should send the larger value" — the backend does
   not; this mismatch is unresolved.
4. **Save**: computes `{CUSTOMER}_{STYLE_ABBREV} {FIT_ABBREV}_SKYNET.dxf`
   (e.g. `ROBERT GATLIN_STRT REG_SKYNET.dxf`) into the output folder — save itself is a TODO
   stub, yet success is returned with this path.

Abbreviations: Straight→STRT, Boot→BOOT, Fashion Boot→FBOOT, Skinny→SKNY, Flare→FLARE;
Regular→REG, Trim→TRIM, Easy→EASY.

Customer-name extraction (lines 67-79) assumes `taskName` like "Order #99340 Robert Gatlin":
takes everything after the first `#`, drops a leading all-digit token. ⚠️ Real task names look
like "Customer Name | Order #" (the work-order endpoint splits on `|`) or "#101378 Name 1/2" —
names containing `/` would produce invalid Windows paths; there is no filename sanitization.

On error: desktop screenshot saved locally as `error_{timestamp}.png` (path reported to the
cloud, file stays on the Windows machine), job marked failed.

## Server-side contract

- `GET /api/optitex/jobs/poll` and `PATCH /api/optitex/jobs/:id` require an
  `Authorization: Bearer` header (401 if missing). The token value is only checked when the
  server's `OPTITEX_AGENT_TOKEN` env var is set (`if (agentToken && token !== agentToken)`,
  routes.ts:792/880) — **unset = any bearer value accepted**.
- Job creation (`POST /api/optitex/jobs`), listing, stats, and **deletion** have no auth at all.
- The only job producer today is the Calculator's "Send to Optitex" button. The webhook pipeline
  does **not** create jobs.

## Setup summary (from OPTITEX_AGENT_SETUP.md)

Windows PC + Optitex 15 + Python 3.10+ → `pip install requests python-dotenv pyautogui keyboard
pillow` → copy `.env.example` to `.env`, set `API_BASE` + `OPTITEX_AGENT_TOKEN` → edit the two
folder constants in source → `python test_connection.py` → `python optitex_agent.py`. Base
pattern naming (`MensBase_41.dxf`) is marked "TBD" in both code and docs. Run as a background
service via Task Scheduler or `pythonw.exe`. While the agent is typing, the machine cannot be
used for anything else (keystrokes go to the focused window).

## Open questions for the team

1. Does a completed (non-TODO) version of the agent exist on the production Windows machine?
   If so, it should be committed — `completed` rows in `optitex_jobs` are otherwise fictitious.
2. What does the agent actually do with the waist range string?
3. Should output .dxf files / error screenshots be uploaded back to the cloud?
4. Is LIFO + ignored priority acceptable, or should the poll order be `priority DESC, createdAt ASC`?
