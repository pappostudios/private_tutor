# Phase 1 plan — marketing site + inquiry form + private admin

Status: **awaiting approval**. Nothing below is built yet.

---

## 1. Architecture

```
Parent's phone ──► static Hebrew page (pre-rendered at build, served from CDN)
                     │  WhatsApp button → wa.me/972XXXXXXXXX?text=<prefilled Hebrew>
                     │  inquiry form (works even before JS loads: Next.js Server Action)
                     ▼
               Server action  submitInquiry()
                 1. honeypot field + "submitted too fast" time-trap  → silently drop bots
                 2. per-IP best-effort limiter (in memory, IP never stored)
                 3. zod validation (same rules as the DB constraints)
                 4. INSERT into Supabase with the *anon* key (RLS: insert-only)
                    └─ DB trigger: global throttle + same-phone cooldown (cannot be bypassed)
                 5. email Erez via Resend (if the DB insert failed, the email carries the
                    full inquiry so the lead is never lost)
                 6. return → thank-you state + WhatsApp fallback button

Erez ──► /admin ──(no session)──► /login  (enter email)
             server checks email === ADMIN_EMAIL, then Supabase signInWithOtp(shouldCreateUser:false)
             email contains a link (/auth/confirm?token_hash=…) AND a 6-digit code
             → session cookie (long-lived, he stays logged in on his phone)
         /admin pages are server-rendered and query Supabase *with Erez's JWT* → RLS decides

Daily scheduled job ──► /api/keepalive (CRON_SECRET) ──► tiny DB call
             (Supabase free projects pause after 7 days without activity — this prevents it)
```

**Stack (as preferred):** Next.js 16 (App Router, TypeScript) + Tailwind v4, Supabase (Postgres + Auth + RLS), Resend for email. Hosting: see §4 — I am flagging a terms-of-service issue with Vercel Hobby and need a decision.

Design choices worth calling out:

- **No service-role key at runtime.** The public form writes with the anon key (RLS insert-only); the admin reads/writes with Erez's own session (RLS admin-only). A leaked server env var therefore can't dump the table unless it's the admin's session.
- **Supabase keys stay server-side** (not `NEXT_PUBLIC_*`). The browser never talks to Supabase directly. RLS is still written as if the anon key were public (defense in depth).
- **Public page is fully static** → Lighthouse ≥ 90 is easy to hold; only the form submission and /admin hit a server.
- **Self-hosted Hebrew font** via `next/font` (Heebo or Assistant, Hebrew subset only), `display: swap`.
- **RTL done with logical properties** (`ms-/me-/ps-/pe-/start/end`), `<html lang="he" dir="rtl">`; directional arrows flipped with `rtl:` variants; brand logos (WhatsApp) not mirrored; phone input is `dir="ltr"` inside the RTL form with right-aligned text (digits read correctly, layout stays RTL).
- **Prices off = not rendered at all** (not CSS-hidden), so amounts never leak into the HTML.
- **Pre-launch guard:** `settings.indexable` defaults to `false` → `noindex` until Erez fills the placeholders, so Google never indexes "[TEXT ABOUT EREZ]". `npm run check:placeholders` lists every remaining placeholder.

### Repo layout

```
content/he.json              ← ALL copy + flags (showPrices, indexable, testimonials[], whatsapp number…)
content/schema.ts            ← zod schema; a broken edit fails the build with a readable error
                               (the previous deploy stays live, so a typo can't take the site down)
src/app/page.tsx             ← main page (static)
src/app/privacy/page.tsx     ← privacy note (text from he.json)
src/app/login/, src/app/auth/confirm/route.ts
src/app/admin/               ← list, status, notes, delete, counts
src/app/api/keepalive/route.ts
src/app/sitemap.ts, robots.ts, opengraph image (static PNG, see §6)
src/lib/phone.ts             ← Israeli phone validate/normalize/format, wa.me link builder
src/lib/supabase/, email.ts, rate-limit.ts
supabase/migrations/0001_init.sql
supabase/tests/rls.sql       ← RLS test suite
tests/e2e/                   ← Playwright: 375px above-the-fold, RTL, form, flags
scripts/og.ts, check-placeholders.ts, verify-rls.ts (smoke test against the live project)
.env.example, README.md
```

---

## 2. Data model

One table, plus one tiny config table. No child name, no IP, no user agent, no cookies/analytics.

```sql
create type inquiry_status as enum ('new','contacted','trial_booked','paying','lost');
create type inquiry_need   as enum ('falling_behind','test_prep','transition_7','confidence','other');
create type inquiry_source as enum ('facebook_group','whatsapp_group','referral','google','notice_board','other');

create table inquiries (
  id                 uuid primary key default gen_random_uuid(),
  created_at         timestamptz not null default now(),

  -- from the form
  parent_first_name  text not null  check (char_length(parent_first_name) between 1 and 40),
  phone              text not null  check (phone ~ '^0(5\d{8}|7\d{8}|[23489]\d{7})$'),  -- normalized, digits only
  grade              smallint not null check (grade between 1 and 9),
  need               inquiry_need not null,
  area               text not null  check (char_length(area) between 1 and 60),
  message            text           check (char_length(message) <= 1000),
  source             inquiry_source not null,
  privacy_version    text not null,                         -- which privacy note they agreed to
  consented_at       timestamptz not null default now(),
  landing_ref        text check (char_length(landing_ref) <= 40),  -- OPTIONAL, see decision D4

  -- admin only
  status             inquiry_status not null default 'new',
  notes              text check (char_length(notes) <= 5000),
  contacted_at       timestamptz,   -- first time each stage was reached;
  trial_booked_at    timestamptz,   -- lets the funnel show "how many ever reached X"
  paying_at          timestamptz,   -- even after an inquiry later becomes "lost"
  lost_at            timestamptz,
  updated_at         timestamptz not null default now()
);
create index on inquiries (created_at desc);

create table admin_users (email text primary key check (email = lower(email)));
```

- **Status pipeline:** `new → contacted → trial_booked → paying`, or `lost` (not a fit / lost) from any stage. A trigger stamps the `*_at` column the first time a stage is reached (and fills earlier stages if one was skipped, e.g. new → paying).
- **Funnel counts** (top of /admin): (a) current count per status, (b) "reached" counts from the `*_at` columns, (c) inquiries per source, plus a small source × outcome table (which groups actually produce paying students). All computed in one SQL query.
- **Deletion:** hard `DELETE` from /admin with a confirm step. Free-tier Supabase has no point-in-time recovery, so a deleted row is gone (only transient logs remain).
- **Phone:** accepts `050-1234567`, `0501234567`, `+972 50 123 4567`, landlines; stored normalized (`0501234567`), shown formatted, WhatsApp link only for mobiles (05X).

---

## 3. Auth + RLS

**Login (single user):**
1. Supabase Auth with **public sign-ups disabled**. Erez's user is created once (invite from the Supabase dashboard).
2. `/login` server action rejects any email ≠ `ADMIN_EMAIL` *before* calling Supabase (generic "a link was sent if allowed" message so the admin email isn't revealed), and calls `signInWithOtp({ shouldCreateUser: false })`.
3. Email template uses `token_hash` links (verified server-side) instead of the PKCE flow, so the link works even if Gmail opens it in a different browser than the one he started in — a common magic-link failure on phones. The same email also shows a 6-digit code as a fallback.
4. `middleware.ts` refreshes the session and redirects `/admin/*` to `/login` when there's none. `/admin` and `/login` are `noindex` and disallowed in robots.txt.

**RLS (the real gate, independent of app code):**

```sql
alter table inquiries enable row level security;
revoke all on inquiries, admin_users from anon, authenticated;

-- anon: INSERT only, only form columns (status/notes/stage dates are not insertable)
grant insert (parent_first_name, phone, grade, need, area, message, source,
              privacy_version, landing_ref) on inquiries to anon;
create policy form_insert on inquiries for insert to anon with check (true);
-- no SELECT/UPDATE/DELETE grant or policy for anon at all

-- admin: everything, but only if the logged-in email is in admin_users
create function is_admin() returns boolean language sql stable security definer
  set search_path = '' as $$
  select exists (select 1 from public.admin_users
                 where email = lower(auth.jwt() ->> 'email'));
$$;
grant select, delete, update (status, notes) on inquiries to authenticated;
create policy admin_select on inquiries for select to authenticated using (is_admin());
create policy admin_update on inquiries for update to authenticated using (is_admin()) with check (is_admin());
create policy admin_delete on inquiries for delete to authenticated using (is_admin());
```

Plus a `BEFORE INSERT` trigger (security definer) that rejects inserts when there were > N inquiries in the last hour globally, or the same phone within the last 10 minutes. This bounds abuse even if someone calls the Supabase API directly, and keeps email volume under Resend's 100/day.

Three independent layers keep a non-allowed email out: the app refuses to send the link; Supabase refuses to create the account (sign-ups off); RLS returns nothing to any JWT whose email isn't in `admin_users`.

**RLS tests** (`supabase/tests/rls.sql`, run in CI via the Supabase CLI and locally against Postgres):
- anon: INSERT ok · SELECT/UPDATE/DELETE denied · cannot set `status`/`notes` on insert
- authenticated non-admin email: sees 0 rows, cannot update/delete
- admin: select/update/delete ok
- constraint checks: bad phone, grade 10, oversized text rejected; throttle trigger fires
- `scripts/verify-rls.ts`: same anon checks against the **live** project after deploy.

**Rotating the admin email:** change `ADMIN_EMAIL` in the host's env vars, run one SQL line (`update admin_users set email = 'new@…'`), invite the new user in Supabase. README will have the exact steps.

---

## 4. Hosting / email / domain options (costs)

Free-tier numbers are as I know them; please verify current terms on each site before signing up.

### Hosting — **decision needed (D1)**

| Option | Cost | Notes |
|---|---|---|
| Vercel Hobby (your preferred) | $0 | Best Next.js support, daily cron included. **Hobby terms restrict it to non-commercial personal use**; a tutoring business site is commercial. Enforcement on tiny sites is rare, but it's a real ToS violation. Pro is $20/month. |
| Netlify Free | $0 | Commercial use allowed. Good Next.js support, scheduled functions included. Credit-based free plan; ample for this site, though each deploy consumes credits (≈ tens of deploys/month). |
| Cloudflare Workers Free (via OpenNext) | $0 | Commercial use allowed, very generous limits, cron triggers free. Slightly more setup; a bit more risk around Next.js feature compatibility. |

**My recommendation:** Netlify Free (ToS-clean, near-zero config). I'll keep the code host-neutral either way (no Vercel-only APIs), so switching later is a redeploy, not a rewrite. If you prefer to stay on Vercel Hobby knowingly, that works technically.

### Database / auth — Supabase Free: $0
500 MB DB (this will use < 1 MB/year), 50k monthly active users. **Pauses after 7 days of no activity** → mitigated by the daily keepalive and by the email fallback. Region: I'd pick **EU (Frankfurt)** — closest to Israel and the EU is an approved destination for transferring personal data out of Israel.

### Email — Resend Free: $0
3,000 emails/month, 100/day. Without a domain, Resend only delivers to the **account owner's own address** from `onboarding@resend.dev` — fine, since notifications only go to Erez, but **Erez must sign up to Resend with the address he wants notifications at**, and should add a "never send to spam" filter. With a custom domain later, sending from e.g. `site@<domain>` becomes possible.
Login emails use Supabase's built-in mailer (free; only sends to project team members and is rate-limited to a few per hour — fine for one user).

### Domain — **Erez's decision (D2)**, nothing is bought

| Option | Approx. cost |
|---|---|
| Free host subdomain (`erez-math.netlify.app` / `.vercel.app` / `.workers.dev`) | $0 |
| `.co.il` via an ISOC-IL accredited registrar | ≈ ₪50–90/year |
| `.com` (e.g. Cloudflare Registrar, sold at cost) | ≈ $10–12/year |

A real domain looks more trustworthy in a WhatsApp preview and is easier to say out loud; it's not required for launch. The site reads its URL from one env var, so adding a domain later is a 10-minute change.

**Total running cost with the recommendation: $0/month** (optional domain only).

---

## 5. Placeholders Erez must fill (all in `content/he.json`)

Marketing copy — every one rendered as an obvious `[…]` placeholder until filled:
1. Brand/display name (just "ארז"? full name? business name?)
2. Hero headline + subheadline
3. About me text; optional photo of Erez (file name in he.json)
4. Method — text for each step (diagnosis lesson → personal plan → regular lessons → short parent update)
5. Who it's for — topics list, **service areas** list
6. How it works — text for the 3 steps (contact → trial lesson → ongoing plan); is the trial lesson free/paid? (business decision)
7. Prices — amounts, lesson length, packages (section hidden until `showPrices: true`)
8. FAQ questions + answers
9. Privacy note text (I'll draft the structure required by Israeli law's notice duty — what's collected, why, who receives it, that it's voluntary, how to delete — as a placeholder; **should be reviewed**, I'm not giving legal advice)
10. SEO title + meta description; OG preview text
11. Testimonials — only real ones, with the parent's permission, no child names (empty by default)

Settings / facts:
12. WhatsApp number (international format) + the prefilled Hebrew message
13. Footer contact details: show phone number? email? (D5)
14. `ADMIN_EMAIL` (login + notifications)
15. Site URL / domain
16. Accent color (I'll pick a neutral, calm palette defined in one place; changeable)

Functional UI text (form labels, validation errors, button labels, the thank-you message) I will write in plain Hebrew so the form works — it's editable in the same file, and makes no promises (e.g. no "I'll reply within an hour").

---

## 6. SEO / sharing

- `<title>`, meta description, canonical, `lang="he" dir="rtl"`, all from he.json.
- **OG image:** a static 1200×630 PNG generated by `npm run og` (renders an HTML template in Chromium → correct Hebrew/RTL; `next/og`'s renderer does not handle RTL text reliably). Kept < 300 KB so WhatsApp shows it. Re-run after changing the name/headline. Note: WhatsApp/Facebook cache previews — use Facebook's Sharing Debugger to refresh.
- schema.org JSON-LD `["LocalBusiness","EducationalOrganization"]` with name, telephone, `areaServed` from he.json. **No ratings, review counts or credentials** are ever emitted.
- `sitemap.xml`, `robots.txt` (disallow `/admin`, `/login`; full `noindex` while `indexable: false`).

---

## 7. How I'll test

- **Unit (vitest):** phone validation/normalization, content schema, funnel-count logic.
- **RLS:** SQL test suite above (CI with Supabase CLI; locally against Postgres with Supabase role shims) + live smoke script.
- **E2E (Playwright, Chromium @ 375×667 and 390×844):** WhatsApp button and form CTA fully inside the first viewport; `dir=rtl` and computed layout direction on sections, form fields, phone input; arrows mirrored; form validation messages; successful submit → thank-you state; testimonials section absent with empty array and present with a fixture; prices absent when flag off and present when on (fixture builds); /admin redirects to /login; status + notes persist; delete removes the row; counts update.
- **Lighthouse (mobile preset)** on the production build: assert performance & accessibility ≥ 90.
- **Secrets:** `.env*` gitignored, `.env.example` documented, a grep check in CI for key-shaped strings.

**Live preview URL:** I can't create accounts on Erez's behalf. I'll build and verify everything locally, then hand over a ~15-minute setup checklist (create Supabase project → run migration; create Resend account; connect the repo to the chosen host; paste env vars). Once those exist, I'll run the live checks (submit → admin → email, RLS smoke test, Lighthouse on the real URL).

---

## 8. Risks

| Risk | Mitigation |
|---|---|
| Supabase free project pauses after 7 idle days → form fails | Daily keepalive job; if insert fails anyway, the email to Erez carries the full inquiry and the parent still sees the thank-you + WhatsApp button |
| Vercel Hobby non-commercial terms | Decision D1; host-neutral code |
| Notification email lands in spam (`onboarding@resend.dev`) | README: add a Gmail filter; custom domain later fixes it properly |
| Magic link opened in a different browser on the phone | `token_hash` flow + 6-digit code fallback |
| Bots bypass the form and hit the API directly | Column-level grants, DB constraints, DB-level throttle; no email is sent for those rows |
| Erez breaks `he.json` while editing | Build-time schema validation; failed build keeps the old version live; README shows how to edit safely in GitHub's web editor |
| Placeholders shared publicly before they're filled | `indexable: false` by default; `check:placeholders` script; I'd recommend not sharing the link until placeholders are filled |
| Free tiers change terms/limits | All three services are swappable; README lists where each is configured |
| Privacy law (Israeli Privacy Protection Law, Amendment 13 in force since Aug 2025) | Minimal data, consent + version stored, deletion, EU region, privacy note structure; **text needs Erez's/a professional's review** |
| Phone number in emails stored at Resend/Gmail | Decision D3 — default to sending no name/phone in the email |
| Dependency updates = maintenance | No Dependabot noise; pinned versions; nothing needs updating to keep running |

---

## 9. Decisions needed before I build

- **D1 Hosting:** Netlify Free (recommended) / Vercel Hobby (preferred stack, non-commercial ToS) / Cloudflare Workers.
- **D2 Domain:** launch on the free subdomain now (recommended), or buy one first — Erez's call.
- **D3 Notification email content:** (a) *recommended* — no name or phone: "New inquiry: grade 7, test prep, Holon, via Facebook group → open admin" (one extra tap; personal data stays out of email); or (b) full details including name + phone.
- **D4 Link tagging:** allow `?src=holon-fb` on shared links, stored in `landing_ref` (shows which specific group post worked, independent of what parents tick). Not personal data. Recommended yes.
- **D5 Public contact details:** show Erez's phone number in the footer (the WhatsApp link exposes it anyway)? Show an email?
- **D6 Retention:** keep inquiries until manually deleted (simplest), or auto-delete `lost` inquiries after N months via a scheduled DB job (better privacy, still zero maintenance). Needs a number if yes.
- **D7 Facts:** OK to state these as fact (not placeholder) in SEO title/schema: "private math tutor, grades 1–9, in-person lessons"? Everything else stays a placeholder.
