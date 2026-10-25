# Ravindu Karunarathne — Education Platform

Next.js 16 (App Router, TypeScript) + Tailwind CSS 4 + Supabase (PostgreSQL, Auth, Storage).

## What is implemented

| Area | Where |
|---|---|
| Public site, 4 course pages, store, contact, privacy/terms | `src/app/(public)` |
| Register / login / forgot + reset password | `src/app/(public)`, `src/app/actions/auth.ts` |
| Student dashboard, profile, packs, payments, settings | `src/app/student` |
| Admin: dashboard, students (+CSV), courses, packs, payments, announcements, Website Editor, settings, audit logs | `src/app/admin/(panel)` |
| All admin mutations (each calls `requireAdmin()`) | `src/app/actions/admin.ts` |
| Secure PDF download (authorise → 60 s signed URL) | `src/app/api/download/[packId]/route.ts` |
| Schema, RLS, atomic Student ID function, payment review | `supabase/migrations/*.sql` |
| Tests | `tests/` |

### Student ID formats
`RKM09`+4 digits (`RKM090001`), `RKM10`+4 (`RKM100001`), `RKM11`+4 (`RKM110001`), `RKPT28`+3 (`RKPT28001`).
*Interpretation note:* the brief says "RKM09 + six-digit sequence" but its examples (`RKM090001`) are 9 characters, i.e. `RKM09` + `0001`. The examples were treated as authoritative. If you really want 6 digits after `RKM09`, change `id_digits` for those courses **before any student registers** (`update courses set id_digits = 6 where id_prefix like 'RKM%'`).
IDs are generated inside the Postgres function `register_student` (row-locked per-course sequence, same transaction as profile + enrollment, unique constraint, immutability trigger). When the Physics range (999) is exhausted, registration is refused with `SEQUENCE_EXHAUSTED`, an audit-log entry is written and the admin dashboard shows a warning. No other format is ever used.

## Setup

1. **Install**: `npm install`
2. **Supabase project** (supabase.com) → copy URL, anon key, service-role key.
3. **Env**: copy `.env.example` → `.env.local` and fill in. Never commit it.
4. **Database**: run the files in `supabase/migrations/` in order (`supabase db push`, or paste each into the SQL editor). This also creates the three storage buckets (`public-assets` public; `study-materials` and `payment-evidence` private).
5. **Auth settings** (Supabase → Authentication):
   - Providers → Email: enabled. **Disable "Allow new users to sign up"** (registration is done server-side; this stops direct API sign-ups).
   - URL configuration: Site URL = your domain; add `<domain>/auth/callback` to Redirect URLs.
   - Configure a custom SMTP provider for reliable reset emails (the built-in sender is heavily rate-limited).
6. **Create the admin** (one-time): set `ADMIN_INITIAL_PASSWORD` in `.env.local` (12+ chars), run `npm run create-admin`, then delete that variable. The admin signs in at `/admin/login`. There is no public admin registration; role is stored in `profiles.role`, which students cannot write (column privileges + RLS).
7. **Run**: `npm run dev` → http://localhost:3000. Then open `/admin/website-editor` to upload your photo/logo, set social links, and `/admin/settings` to enter your bank details.

## Deploy (Vercel)
Import the repo, add the env vars from `.env.example` (service-role key and Turnstile secret as server-only), set `NEXT_PUBLIC_SITE_URL` to the production URL, deploy. Server Actions accept uploads up to 50 MB (`next.config.ts`); Vercel's own request-body limits for your plan also apply — for large PDFs check them.

## Tests
- `npm test` runs unit tests plus `tests/sql.test.ts`, which starts a throw-away embedded PostgreSQL, applies the real migrations and verifies IDs, concurrency (25 parallel registrations), rollback, exhaustion, immutability, RLS, privilege escalation and the payment workflow.
- `tests/db.integration.test.ts` runs the same style of checks against a real Supabase project; it is skipped unless real keys are in `.env.local` (use a dev project).
- `npm run lint` = TypeScript check.

## Still requires your configuration / not verified here
- Supabase project, Auth settings, SMTP (password-reset email) — **not tested end-to-end** (no project was available).
- Storage uploads/signed URLs, Supabase Auth sign-in, and the browser UI were **not run against a live Supabase**; the app compiles (`next build` passes) and the database layer is tested on local Postgres.
- Contact form stores messages in the admin Settings page; **it does not send email**.
- Registration marks emails as confirmed without sending a confirmation email (to avoid depending on SMTP). Enable verification later if you want proof of mailbox ownership.
- Optional Cloudflare Turnstile: server verification is implemented (`verifyBot`); the widget is not yet added to the forms. Rate limiting and honeypots are active.
- Privacy Policy / Terms are templates; have them reviewed. No bank details, testimonials or statistics are invented.
- Online payment gateway: not implemented. `review_payment` is the single approval path, so a verified gateway callback can call it later.
- Font settings are fixed (Inter + Noto Sans Sinhala); not editable from the admin.
- Responsive layouts were built mobile-first but not yet checked on real devices/browsers.
