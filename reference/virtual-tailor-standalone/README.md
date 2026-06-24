# bdj-measurements

Standalone Virtual Tailor measurement-capture page for Blue Delta Jeans. Customers who skip the in-checkout Virtual Tailor on the Shopify storefront fill this form post-purchase; the pattern team reads submissions from Supabase and replays them through their own Bold Metrics request.

This is a stop-gap until a real internal database exists. It captures the same Virtual Tailor question set as the storefront form (`../snippets/virtual-tailor-3.liquid`, `../assets/VirtualTailor.js`) — but it does **not** call the Bold Metrics API and does **not** send anything to Klaviyo.

## Stack

- Next.js 16 (App Router) + TypeScript
- Supabase (Postgres + RLS) for storage
- zod for validation (server authoritative)
- Upstash Redis for IP rate limiting (optional)
- Vercel for hosting

## Local development

```bash
cp .env.example .env.local
# fill in NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY,
# SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY

npm run dev
# open http://localhost:3000
```

## Supabase setup

1. Create a Supabase project (any region).
2. Apply `supabase/migrations/0001_virtual_tailor_submissions.sql` via Studio's SQL editor or the Supabase CLI.
3. Populate the project URL, anon key, and service-role key into `.env.local` from the Supabase project's API settings.
4. Add additional pattern-team email addresses to both RLS policies in the migration (`admin_select`, `admin_update_processed`) before going live.

To verify RLS is working, paste this in the SQL editor:

```sql
-- As an anon role (simulating the browser), should return zero rows
set role anon;
select count(*) from public.virtual_tailor_submissions;
-- expect: ERROR: permission denied
```

## Pattern team workflow (Phase 1 — Studio only)

1. Pattern-team emails sign in to Supabase Studio.
2. Open the **virtual_tailor_submissions** table.
3. Filter `processed = false`, sort by `created_at desc`.
4. Each row's `bold_metrics_payload` (jsonb) is already shaped for the Bold Metrics API — copy it into your existing request, append your `user_key`, and POST to `https://api.boldmetrics.io/virtualtailor/get`.
5. Once measured, set `processed = true`, `processed_at = now()`, `processed_by = '<your email>'`.

## Spam / abuse protection

- **Honeypot field** (`company_website`) — bots fill it; humans never see it. Server rejects any submit where it's non-empty.
- **Min time on form** (~3s) — server rejects submits faster than this.
- **IP rate limit** (5 / minute) — sliding window via Upstash. Without `UPSTASH_REDIS_REST_URL` set, the limiter is a no-op.

If abuse appears, the next escalation is Cloudflare Turnstile in front of the submit button.

## Deploy to Vercel

```bash
# from this directory
vercel --prod
```

Set these env vars in the Vercel project (Production + Preview):

| Var | Notes |
|-----|-------|
| `NEXT_PUBLIC_SUPABASE_URL` | public URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | public anon key (Phase 2 admin only) |
| `SUPABASE_URL` | server mirror |
| `SUPABASE_SERVICE_ROLE_KEY` | **server-only**; never `NEXT_PUBLIC_*` |
| `UPSTASH_REDIS_REST_URL` | optional |
| `UPSTASH_REDIS_REST_TOKEN` | optional |

## Architecture notes

- `lib/steps.ts` is the single source of truth: option arrays, bounds, gender-specific sequences, height math, and the `getBraSize` join. Lifted from the Shopify implementation so behavior parity is one diff away.
- `lib/bold-metrics.ts` mirrors `generateBoldMetricsUrl()` in `assets/VirtualTailor.js:2661-2709`, minus the `user_key` (the pattern team re-attaches their key on replay). The output object goes into `bold_metrics_payload` jsonb verbatim.
- `app/actions.ts` is the only path into the database. The browser never sees the service-role key. Validation runs against `lib/schema.ts` (zod) before any insert.
- `components/MeasurementWizard.tsx` holds all wizard state (current step index, form data, submit status). Steps are rendered conditionally by `currentStepId`. Form draft is persisted to `localStorage` (`bdjFormData`) so a reload doesn't lose progress.

## Verification checklist

- [ ] Walk the male path end-to-end → row in Supabase with `gender='m'`, `waist_circum_preferred` populated, `bra_size` null.
- [ ] Walk the female path end-to-end → row with `gender='w'`, `bra_size='34B'` (or similar), `waist_circum_preferred` null.
- [ ] `bold_metrics_payload` matches the URL params that `generateBoldMetricsUrl()` would produce for the same inputs.
- [ ] Anon `select` from the browser is denied.
- [ ] Honeypot field non-empty → submit rejected.
- [ ] >5 submits/min from the same IP → 429 / rejection (only with Upstash configured).

## Phase 2 — Pattern Team Dashboard

Two new routes in the same project:

- **`/team`** — Mobile-first dashboard for the pattern team. Card list of unprocessed submissions, search by email/name/order, tap to open a drawer with full measurements + a one-click "Copy Bold Metrics URL params" button + "Mark Processed". No test/junk rows visible.
- **`/admin`** — Desktop-first admin dashboard, only `dev@bluedeltajeans.com` (the `admin` role) can access. Filterable table with status chips (New/Processed/Test), CSV export, "Mark as Test" toggle. `/admin/users` manages the allow-list and per-user notification prefs.

### Auth

Google OAuth as the primary sign-in path (configured in Supabase → Auth → Providers → Google). Magic-link email as fallback. Both flows pass through `/auth/callback` to set the session cookie.

Allow-list enforced at TWO layers: `middleware.ts` checks `user_profiles` and bounces unknown emails to `/login?error=unauthorized`; RLS + SECURITY DEFINER RPCs enforce the same rules at the DB layer so a bypassed middleware can't leak data.

### Notifications

On customer submit, every user with `notify_on_new_submission = true` gets a Resend email summarizing the submission with a button into `/team`. Each user can toggle their own preference from the Settings drawer (`update_my_notification_pref` RPC). Notification failure is logged but never fails the customer's submit.

Required env vars for Phase 2:
- `RESEND_API_KEY` — server-only. Without it, notifications silently skip.
- `NEXT_PUBLIC_SITE_URL` — used for the dashboard link in emails.

### Migration

`supabase/migrations/0002_phase2_users_and_test_flag.sql`: adds the `user_profiles` table, an `is_test` column on submissions, role-aware RLS policies, and 5 SECURITY DEFINER RPC functions (`mark_processed`, `update_my_notification_pref`, `admin_mark_test`, `admin_upsert_user`, `admin_remove_user`). Additive — does not break the existing public form.

---

## See also

- **Hub doc — Virtual Tailor:** [`../../03-virtual-tailor.md`](../../03-virtual-tailor.md)
  — what the Virtual Tailor is, how **both** implementations (this standalone app + the in-theme
  Shopify form) work, the customer journey, and the parity rule that keeps `lib/steps.ts`,
  `lib/schema.ts`, and `lib/bold-metrics.ts` in sync with the theme's `virtual-tailor-3.liquid` +
  `VirtualTailor.js`.
- **Bold Metrics payload contract:** see the hub doc's
  [§6 Bold Metrics payload contract](../../03-virtual-tailor.md#6-bold-metrics-payload-contract)
  for the endpoint, the full input list (incl. the gender split and `sleeve_type=ARS`), `user_key`
  handling, the six outputs, the `Waist Avg.` 3-way mean, and `Thigh Circum = thigh_circum_proximal`.
  This is the contract `lib/bold-metrics.ts` must honor.
