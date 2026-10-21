# Phase 1 plan — marketing site + inquiry form + private admin

Status: **rev 2.** Sections 4–7 approved (rev 1, decisions folded in). Sections 1–3 rewritten for Cloudflare, **awaiting review**. Nothing is built yet.

---

## 0. Host: Cloudflare Workers (Free plan)

- **Commercial use:** Cloudflare's Self-Serve Subscription Agreement (last updated Sept 2025) has no non-commercial clause. Vercel Hobby has an explicit one. Cloudflare's own plans FAQ presents the Free plan as fine for websites, and community answers confirm business sites are fine on Free. The only service-specific restriction I found concerns serving video, which doesn't apply here.
  - *Caveat:* I couldn't open cloudflare.com from this build sandbox (blocked by its network policy), so this comes from search results quoting the terms, not my own reading. Erez should skim the "Restrictions" section of https://www.cloudflare.com/terms/ when he signs up.
- **Free-plan limits that matter:**
  - Static-asset requests are **free and unlimited**.
  - Requests that run Worker code: 100,000/day, **10 ms CPU** each.
  - 5 cron triggers per account.
  - Workers Builds (deploy from GitHub): 3,000 build minutes/month.
- **URL:** `<name>.<account>.workers.dev` for build/preview. A `.co.il` can be attached before launch (custom domains are free; DNS can move to Cloudflare for free). The site URL lives in one env var (`SITE_URL`).
- **No SLA** on the Free plan. That's acceptable for this site.

---

## 1. Architecture

### Why the shape changed from rev 1
On Workers Free, every request that runs server code is capped at **10 ms CPU** and counts toward the 100k/day limit. Static files cost nothing and have no limit. Running the full Next.js server on Workers (via the OpenNext adapter) would send every page view through Worker code. It would also be the most fragile part of the stack.

So:
- **Next.js 16 + Tailwind v4 with `output: 'export'`:** the public site and the admin UI are prebuilt static files.
- **One small Worker** (plain TypeScript, ~200 lines) handles only `/api/*` plus a daily cron.

**Stated deviation from the preferred stack:** Next.js is still the framework, but without Next's server features (Server Actions, middleware, server-side rendering). The reason is the 10 ms CPU limit and lower maintenance on the chosen host.

```
PARENT'S PHONE
  GET /  /privacy  /og.png …  ─► Cloudflare static assets (prebuilt; free; no Worker run)
  WhatsApp button             ─► wa.me/972XXXXXXXXX?text=<prefilled message from he.json>
  POST /api/inquiry           ─► WORKER
                                   1. honeypot field + time-trap (submitted < 3 s after load)
                                      → bot gets a fake "thanks", nothing stored or sent
                                   2. per-IP limit (Cloudflare rate-limit binding; IP never stored)
                                   3. zod validation (same rules as the DB constraints)
                                   4. Supabase RPC submit_inquiry(data, FORM_KEY)  ← the ONLY write path
                                   5. email Erez via Resend
                                      (if step 4 failed, this email carries the full inquiry → lead not lost)
                                   6. JSON → thank-you state with WhatsApp fallback
                                      (no-JS: plain form POST → 303 redirect to /?sent=1#contact)

EREZ'S PHONE
  GET /admin                  ─► static shell: no data, no secrets → shows login screen if no session
  POST /api/admin/login       ─► WORKER: email == ADMIN_EMAIL ?
                                   yes → Supabase signInWithOtp(shouldCreateUser: false)
                                   always → same generic reply ("if allowed, a link was sent")
  email: link /auth/confirm?token_hash=… AND a 6-digit code ─► session stored in his browser
  list / status / notes / delete ─► browser → Supabase directly, with Erez's JWT → RLS is_admin()

CRON (Worker scheduled handler, daily) ─► Supabase keepalive()
  (Supabase Free pauses a project after 7 days without activity)
```

### Key design choices
- **No Supabase secret / service-role key anywhere.** The Worker holds a `FORM_KEY` that can only insert inquiries. The admin uses Erez's own session. The worst a leaked Worker secret allows is inserting rows.
- **The Supabase publishable (anon) key is in the browser bundle** (needed for the admin login). This is by Supabase's design. The database is configured on the assumption that the key is public: anon can't touch any table directly (see §3).
- **The `/admin` shell is public; the data is not.** The HTML/JS of the admin page holds no data and no secrets. Everything is fetched after login and gated by RLS. *Optional hardening (not in default scope):* put Cloudflare Access (free, email one-time code) in front of `/admin*`, so the shell itself isn't served without a login. That adds a second login step for Erez. → **review point R1**
- **Worker CPU stays far under 10 ms:** the Worker mostly waits on network calls (Supabase, Resend), and waiting doesn't count toward the limit.
- **Deploys:** Workers Builds, connected to GitHub. A push to `main` deploys production; other branches get preview URLs. Copy edits made in GitHub's web editor go live in about 2 minutes.
- **Other choices carried over from rev 1:**
  - The content file is checked at build time; a broken edit fails the build and the old version stays live.
  - Hebrew font is self-hosted.
  - RTL uses logical properties.
  - Prices aren't rendered at all when the flag is off.
  - The site is `noindex` until `indexable: true`.
  - A script lists any placeholders still unfilled.

### Repo layout
```
content/he.json            ← ALL copy + flags (showPrices, indexable, testimonials[], whatsapp…)
content/schema.ts          ← zod schema for he.json
src/app/page.tsx           ← main page (static)
src/app/privacy/page.tsx   ← privacy note
src/app/admin/page.tsx     ← admin SPA (client-side; login view + dashboard)
src/app/auth/confirm/page.tsx
src/app/sitemap.ts, robots.ts
src/components/…           ← sections, form, admin pieces
src/lib/phone.ts           ← IL phone validate / normalize / format / wa.me link (shared by form, Worker, admin)
src/lib/inquiry-schema.ts  ← zod schema shared by form + Worker
src/lib/funnel.ts          ← count logic (unit-tested)
worker/index.ts            ← /api/inquiry, /api/admin/login, scheduled keepalive
wrangler.jsonc             ← assets = ./out, run_worker_first = ["/api/*"], cron, rate-limit binding
supabase/migrations/0001_init.sql
supabase/tests/rls.test.sql
tests/unit, tests/e2e      ← vitest, Playwright
scripts/og.ts, check-placeholders.ts, verify-rls.ts
.env.example, README.md
```

### Environment variables
| Var | Where | Secret? |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | build (browser) | no (public by design) |
| `NEXT_PUBLIC_SITE_URL` | build | no |
| `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY` | Worker | no |
| `FORM_KEY` | Worker secret (its SHA-256 hash is stored in the DB) | **yes** |
| `ADMIN_EMAIL` | Worker secret | mildly |
| `RESEND_API_KEY` | Worker secret | **yes** |
| `NOTIFY_EMAIL_TO` | Worker (defaults to `ADMIN_EMAIL`) | mildly |

---

## 2. Data model

One data table, an admin allow-list, and a private settings table. Nothing about the child beyond their grade. No IP, user agent, cookies or analytics.

```sql
create type inquiry_status as enum ('new','contacted','trial_booked','paying','lost');
create type inquiry_need   as enum ('transition_7','test_gaps','falling_behind','confidence','other');
create type inquiry_source as enum ('facebook_group','whatsapp_group','referral','google','notice_board','other');

create table public.inquiries (
  id                 uuid primary key default gen_random_uuid(),
  created_at         timestamptz not null default now(),

  -- from the form
  parent_first_name  text not null check (char_length(btrim(parent_first_name)) between 1 and 40),
  phone              text not null check (phone ~ '^0(5\d{8}|7\d{8}|[23489]\d{7})$'),  -- normalized digits
  grade              smallint not null check (grade between 1 and 9),
  need               inquiry_need not null,
  area               text not null check (char_length(btrim(area)) between 1 and 60),
  message            text check (char_length(message) <= 1000),
  source             inquiry_source not null,
  privacy_version    text not null check (char_length(privacy_version) <= 20),
  consented_at       timestamptz not null default now(),
  landing_ref        text check (landing_ref ~ '^[a-z0-9-]{1,40}$'),   -- only if D4 = yes

  -- admin only
  status             inquiry_status not null default 'new',
  notes              text check (char_length(notes) <= 5000),
  contacted_at       timestamptz,     -- first time each stage was reached
  trial_booked_at    timestamptz,
  paying_at          timestamptz,
  lost_at            timestamptz,
  updated_at         timestamptz not null default now()
);
create index inquiries_created_at_idx on public.inquiries (created_at desc);
create index inquiries_phone_created_idx on public.inquiries (phone, created_at desc);

create table public.admin_users (email text primary key check (email = lower(email)));

create schema private;                              -- NOT exposed through the Supabase API
create table private.settings (
  id               boolean primary key default true check (id),   -- single row
  form_key_sha256  bytea not null
);
```

**Notes:**
- **"Help with" options:** labels and display order come from `he.json`, with "מעבר לכיתה ז'" and "השלמת פערים לפני מבחן" first. The enum keys are stable, so reordering or relabeling never needs a migration. `test_prep` from rev 1 is renamed `test_gaps` to match the new label.
- **Area field:** service area is Tel Aviv only, so it's labeled "שכונה" (free text). A parent from outside Tel Aviv can still write their city, and Erez decides.
- **Status pipeline:** `new → contacted → trial_booked → paying`, or `lost` from any stage. A `BEFORE UPDATE` trigger stamps the matching `*_at` column the first time a stage is reached. It also back-fills skipped earlier stages (e.g. new → paying), and sets `updated_at`. "Lost" keeps the earlier stamps, so the funnel still shows how far an inquiry got.
- **Counts shown at the top of /admin:**
  - current count per status
  - "ever reached" count per stage, taken from the `*_at` columns
  - total per source
  - a source × outcome table
  - These are computed in the browser from the fetched rows by `src/lib/funnel.ts` (unit-tested); a database view isn't worth it at this volume.
- **Deletion:** a permanent `DELETE` with a confirm step. Supabase Free has no point-in-time recovery.
- **Phone:** accepts `050-1234567`, `0501234567`, `+972 50 123 4567`, `972501234567`, and landlines. It's stored normalized and shown formatted. The WhatsApp link appears only for mobile numbers (05X).

---

## 3. Auth + RLS

### Login (single user)
1. Supabase Auth with **sign-ups disabled**. Erez's account is created once by an invite from the dashboard.
2. `/api/admin/login` (Worker) compares the email to `ADMIN_EMAIL`, case-insensitively.
   - If it matches, it calls Supabase OTP with `shouldCreateUser: false`.
   - If not, it does nothing.
   - The reply is identical either way, so the page doesn't reveal which email is the admin's.
   - A per-IP rate limit applies.
3. **Email template:**
   - The link is `{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=email`, verified with `verifyOtp`. This works even when Gmail opens the link in a different browser than the one where he asked for it (the PKCE flow breaks in that case).
   - The same email shows `{{ .Token }}` as a 6-digit code he can type in instead.
4. **Session:** stored in his browser and refreshed automatically, so he stays logged in on his phone. A "log out" button signs out of every device.
5. **Supabase's built-in mailer only sends to the project's team members**, so `ADMIN_EMAIL` must be the address Erez signs up to Supabase with. After the domain is bought, login emails can switch to Resend SMTP.

**Why a non-allowed email can't log in** (three independent layers):
- the Worker refuses to request a link;
- Supabase refuses to create an account (sign-ups are off);
- even an account that somehow exists sees **zero rows**, because its email isn't in `admin_users`.

### Database privileges + RLS
The publishable key is assumed public. Anon therefore gets **no table privileges at all**. The only way to insert is an RPC that needs `FORM_KEY`, which only the Worker holds.

```sql
-- Lock everything down first (Supabase grants anon/authenticated broad defaults on public)
alter table public.inquiries   enable row level security;
alter table public.admin_users enable row level security;
revoke all on public.inquiries, public.admin_users from anon, authenticated, public;
revoke all on schema private from anon, authenticated, public;

-- Admin check: helper lives in `private`, so it isn't callable via the API
create function private.is_admin() returns boolean
  language sql stable security definer set search_path = '' as $$
  select exists (select 1 from public.admin_users
                 where email = lower(coalesce(auth.jwt() ->> 'email', '')));
$$;
revoke all on function private.is_admin() from public;
grant usage on schema private to authenticated;
grant execute on function private.is_admin() to authenticated;

-- Admin: read, change status/notes only, delete
grant select, delete on public.inquiries to authenticated;
grant update (status, notes) on public.inquiries to authenticated;
create policy admin_select on public.inquiries for select to authenticated using (private.is_admin());
create policy admin_update on public.inquiries for update to authenticated
  using (private.is_admin()) with check (private.is_admin());
create policy admin_delete on public.inquiries for delete to authenticated using (private.is_admin());
-- No INSERT for authenticated; no policies at all on admin_users (managed in the SQL editor)

-- The only write path for the public form
create function public.submit_inquiry(form_key text, p jsonb) returns void
  language plpgsql security definer set search_path = '' as $$
begin
  -- 1. caller must hold FORM_KEY (compared by SHA-256)
  -- 2. throttle: reject if > 20 inquiries in the last hour or > 60 in the last day (global)
  -- 3. duplicate guard: same phone within 10 minutes → return quietly (no second row)
  -- 4. insert ONLY the form columns from p; status/notes/stage dates are never taken from input
end $$;
revoke all on function public.submit_inquiry(text, jsonb) from public;
grant execute on function public.submit_inquiry(text, jsonb) to anon;

create function public.keepalive() returns timestamptz language sql stable as $$ select now() $$;
revoke all on function public.keepalive() from public;  grant execute on function public.keepalive() to anon;
```

**How this maps to the acceptance criterion** ("anon cannot read or write the inquiries table except INSERT via the form"):
- Anon cannot `SELECT`, `INSERT`, `UPDATE` or `DELETE` the table directly. Not even direct `INSERT` works, so nobody can skip the honeypot and rate limit by calling Supabase with the public key.
- The form's insert goes through `submit_inquiry`, and only with the Worker's `FORM_KEY`.
- This is stricter than an "anon may INSERT" policy, and I think it's the right reading of "INSERT via the form". → **review point R2**
- **Alternative I rejected:** have the Worker insert with the Supabase secret (service-role) key. That works, but the key bypasses all RLS, so a leak would expose every inquiry.

**Throttle numbers:**
- 20/hour and 60/day globally, tuned to Resend Free's 100 emails/day and far above any real volume.
- When the throttle trips, the parent sees a polite error with the WhatsApp button.
- The numbers are constants at the top of the migration.

### RLS test suite (`supabase/tests/rls.test.sql`)
**Runs:**
- in CI (GitHub Actions + Supabase CLI local stack);
- in this sandbox against plain Postgres with Supabase's `anon` / `authenticated` roles and `auth.jwt()` emulated;
- `scripts/verify-rls.ts` repeats the anon checks against the **live** project after setup.

**Checks:**

| As | Check | Expected |
|---|---|---|
| anon | `select` / `insert` / `update` / `delete` on `inquiries` | denied |
| anon | `select` on `admin_users` | denied |
| anon | `private.is_admin()` | not callable |
| anon | `submit_inquiry` with a wrong or missing key | error, no row |
| anon | `submit_inquiry` with the right key | exactly 1 row; status `new`, notes null, even if `p` tries to set them |
| anon | invalid phone, grade 0 or 10, oversized text | rejected |
| anon | throttle and duplicate guard | fire |
| authenticated, non-admin email | `select` | 0 rows |
| authenticated, non-admin email | `update` / `delete` | 0 rows affected |
| admin | `select` / `update(status, notes)` / `delete` | work |
| admin | updating `phone` or `created_at` | denied (column grant) |
| admin | status trigger | stamps stage dates correctly, including skipped stages and lost-after-paying |

### Rotating the admin email
1. Invite the new address in Supabase Auth.
2. Run `update admin_users set email = 'new@…';`.
3. Change `ADMIN_EMAIL` in Cloudflare.
4. Delete the old Supabase user.

The README will have copy-paste steps.

---

## 4. Hosting / email / domain (approved; updated with decisions)

| Service | Plan | Cost | Notes |
|---|---|---|---|
| Cloudflare Workers + static assets + Workers Builds | Free | $0 | Commercial use OK (see §0). Previews on `*.workers.dev`. |
| Supabase | Free | $0 | EU (Frankfurt) region. Pause after 7 idle days, handled by the daily keepalive. |
| Resend | Free | $0 | 3,000/month, 100/day. Before the domain exists, it only delivers to the account owner's address, so Erez signs up to Resend with his notification address. After the `.co.il` is verified, mail comes from his domain and is less likely to be marked as spam. |
| Domain | — | build on `workers.dev`; Erez buys a `.co.il` (≈ ₪50–90/yr) before public launch | Attach as a Cloudflare custom domain, then change `SITE_URL`. |

Running cost: **$0/month** plus the domain.

## 5. Content (approved; updated with decisions)

**Decided copy, used as-is:**
- **Display name:** "ארז [שם משפחה] · מורה פרטי למתמטיקה" (the last name stays a placeholder).
- **Hero headline:** "מעבר בטוח לחטיבה — בלי פערים במתמטיקה".
- **Hero subheadline:** as given (grades 5–8, Tel Aviv, diagnosis → close gaps → confidence, before 7th grade and tests).
- **Secondary line:** "מלמד גם כיתות א'–ט'".
- **CTAs:** "דברו איתי בוואטסאפ" / "השאירו פרטים ואחזור אליכם".
- **WhatsApp prefilled message:** as given.
- **Service area:** "תל אביב" in copy, SEO meta and schema.org `areaServed`.
- **First lesson:** "שיעור אבחון", paid, with "if it's not a fit, you don't pay"; the price stays a placeholder.
- **"Help with" order:** "מעבר לכיתה ז'", then "השלמת פערים לפני מבחן", then the rest.
- **Footer:** WhatsApp + phone only. No public email.

**Placeholders Erez still fills:** last name, about-me text, photo, FAQ, prices, WhatsApp/phone number, `ADMIN_EMAIL`.

**Placeholders I'll keep for sections the brief didn't cover:**
- method-step details;
- "who it's for" topics list;
- how-it-works step text beyond the decided diagnosis-lesson terms.

Each is clearly marked `[…]`. **Privacy note:** structure drafted and marked **"טיוטה — דורש בדיקה / needs review"**.

**Accent color:** calm teal/blue on warm off-white, with WCAG AA contrast; defined in one place.

## 6. SEO / sharing (approved)
- Title, description and canonical come from he.json, with a Tel Aviv focus.
- Static 1200×630 OG image rendered in Chromium (handles Hebrew correctly).
- JSON-LD `["LocalBusiness","EducationalOrganization"]` with `areaServed: תל אביב`; no ratings, reviews or credentials.
- Sitemap; robots.txt disallows `/admin` and `/auth`; `noindex` until `indexable: true`.

## 7. Testing (approved; host-adjusted)
- **Unit:** vitest.
- **RLS:** suite in §3.
- **End-to-end:** Playwright at 375 px and 390 px. Checks:
  - both CTAs visible without scrolling;
  - RTL layout throughout;
  - form validation and the thank-you state;
  - the testimonials and prices toggles;
  - admin login gate, status, notes and delete;
  - the counts.
- **Local Worker:** run with `wrangler dev`.
- **Lighthouse:** mobile preset, performance and accessibility ≥ 90.
- **Secrets:** a scan in CI.
- **Live preview:** this sandbox can't reach Cloudflare's API, so deploys happen through Workers Builds after Erez connects the repo (setup checklist in the README, ~15 min). I'll then run the live checks on the preview URL.

## 8. Risks (approved; additions marked ★)
- Supabase pause → keepalive + email fallback.
- Notification email in spam until the domain exists.
- Magic link opened in a different browser → `token_hash` + 6-digit code.
- Broken he.json → build fails safely.
- Placeholders shared too early → `noindex` + checker.
- Free-tier terms change → every service is swappable.
- Privacy law (Amendment 13) → minimal data, consent version stored, deletion, EU region, note marked for review.
- ★ Workers Free has no SLA, and a Worker can't use more than 10 ms CPU per request. Only 2 small endpoints run code, so this is fine at this scale.
- ★ The Cloudflare rate-limit binding is per Cloudflare location, i.e. approximate. The DB throttle is the hard backstop.
- ★ Commercial-use reading of Cloudflare's terms is from secondary sources (see §0) → Erez checks the terms at signup.

## 9. Still-open decisions
- **D3 — notification email content.** Recommended: no name or phone in the email; he taps through to /admin. The DB-failure fallback email always carries full details.
- **D4 — `?src=` link tagging into `landing_ref`.** Recommended: yes.
- **D6 — retention.** Manual delete only, or auto-delete `lost` inquiries after N months (a daily job, no maintenance).
- **R1 — Cloudflare Access in front of `/admin`.** Recommended: no for now; RLS already protects the data.
- **R2 — insert via a keyed RPC instead of an anon INSERT policy.** Recommended: yes.
