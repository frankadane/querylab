# QueryLab

Online and live training academy for AI, Data Science (Python/SQL) and Power BI.

**Stack:** Next.js 15 (App Router) · TypeScript · Tailwind CSS v4 · shadcn/ui (Radix) ·
Supabase · Resend · Vercel.

## Status

| Phase | Scope                                                               | State   |
| ----- | ------------------------------------------------------------------- | ------- |
| 1     | Foundation: tooling, design tokens, layout, dark mode, landing page | ✅ done |
| 2     | Auth & profiles                                                     | ⏳ next |
| 3     | Courses & CMS                                                       |         |
| 4     | Live classes                                                        |         |
| 5     | Learning experience                                                 |         |
| 6     | Payments & polish                                                   |         |

## Getting started

Requirements: Node 20+, pnpm 10, Docker (for local Supabase).

```bash
pnpm install
pnpm db:start            # starts local Supabase in Docker (first run pulls images)
pnpm db:reset            # applies migrations + seeds demo users
cp .env.example .env.local
```

Fill `.env.local` from `npx supabase status -o env`:
`API_URL` → `NEXT_PUBLIC_SUPABASE_URL`, `ANON_KEY` → `NEXT_PUBLIC_SUPABASE_ANON_KEY`,
`SERVICE_ROLE_KEY` → `SUPABASE_SERVICE_ROLE_KEY`. Then:

```bash
pnpm dev
```

| URL                    | What                                                                                |
| ---------------------- | ----------------------------------------------------------------------------------- |
| http://localhost:3000  | the app                                                                             |
| http://127.0.0.1:54324 | **Mailpit**: every auth email (confirm, magic link, reset, email change) lands here |
| http://127.0.0.1:54323 | Supabase Studio (database browser)                                                  |

App emails (e.g. "account deleted") print to the terminal running `pnpm dev` until `RESEND_API_KEY` is set.

### Demo accounts (local only)

Seeded by `supabase/seeds/01_users.sql`, flagged `is_demo = true`. **Never seed these in production.**

| Role       | Email                    | Password                |
| ---------- | ------------------------ | ----------------------- |
| Admin      | admin@querylab.test      | `Demo-Admin-2026!`      |
| Instructor | instructor@querylab.test | `Demo-Instructor-2026!` |
| Student    | student1@querylab.test   | `Demo-Student-2026!`    |
| Student    | student2@querylab.test   | `Demo-Student-2026!`    |

## Scripts

| Command                             | What it does                                                                       |
| ----------------------------------- | ---------------------------------------------------------------------------------- |
| `pnpm dev`                          | Dev server                                                                         |
| `pnpm build` / `pnpm start`         | Production build / serve                                                           |
| `pnpm lint` · `pnpm typecheck`      | ESLint · TypeScript                                                                |
| `pnpm format` · `pnpm format:check` | Prettier                                                                           |
| `pnpm test`                         | Unit tests (Vitest)                                                                |
| `pnpm test:e2e`                     | Playwright + axe accessibility tests at 375/768/1024/1440 (run `pnpm build` first) |

## Project layout

```
src/app/(marketing)/   public pages (landing + placeholders for later phases)
src/components/site/   header, footer, nav, theme toggle, logo
src/components/landing/ landing-page sections (animated hero dashboard)
src/components/ui/     shadcn/ui primitives
src/app/(auth)/        login, register, magic link, forgot/reset password + server actions
src/app/(app)/         signed-in area: dashboard, account, admin + instructor (role-gated)
src/app/auth/confirm   handles every email link (token-hash verification)
src/lib/auth.ts        getCurrentUser / requireUser / requireRole (role read from DB)
src/lib/supabase/      server, admin (service role) and middleware clients; generated types
src/lib/validation/    zod schemas shared by client forms and server actions
src/lib/rate-limit.ts  per-IP + per-account limits backed by Postgres
src/lib/email/         Resend sender (console fallback)
src/lib/site.ts        site config, navigation, tracks
supabase/migrations/   SQL schema, RLS policies
supabase/templates/    auth email templates
supabase/tests/        pgTAP RLS tests
docs/design-system.md  palette, type, contrast results, design decisions
docs/deployment.md     how to deploy to Vercel + Supabase
docs/payments.md       order lifecycle, providers, adding Stripe/PayPal
tests/                 unit (Vitest) and e2e (Playwright)
```

## Payments, content, analytics, SEO (Phase 6)

- **Payments** (no Stripe at launch): bank transfer / invoice, free, 100% coupon and admin-recorded
  payments through a `PaymentProvider` interface. Orders, coupons, PDF invoices (pro-forma while
  pending), refunds that revoke access. **See `docs/payments.md`**, including how to add Stripe or PayPal
  without schema changes.
- **Blog & pages CMS** (Admin → Blog / Pages): Markdown with preview, image upload (alt text required),
  SEO fields. About, Privacy, Terms and Cookie notice are editable _templates_, so get them reviewed.
- **Enquiries:** contact and corporate-training forms (honeypot + rate limit) → Admin → Enquiries.
- **Analytics** (Admin → Analytics): sign-ups and enrolments per week, revenue recorded per currency,
  live-session attendance rate and completion by course.
- **SEO:** metadata and OpenGraph on every page, generated social image, `sitemap.xml`, `robots.txt`,
  JSON-LD (`EducationalOrganization`, `Course`, `Event`, `Article`); private pages are `noindex`.
- **Deployment:** step-by-step guide in **`docs/deployment.md`**.

### Quality checks

| Check                                                         | Result                                                            |
| ------------------------------------------------------------- | ----------------------------------------------------------------- |
| Lighthouse (mobile): home, catalogue, course, schedule, blog  | Performance 92–97, Accessibility 100, Best practices 100, SEO 100 |
| axe (WCAG 2.2 AA) on every page, light + dark, 375px + 1440px | 0 violations, no horizontal scrolling                             |
| pgTAP RLS / function tests                                    | 156                                                               |
| Unit (Vitest)                                                 | 69                                                                |
| Integration (Supabase API)                                    | 16                                                                |
| End-to-end (Playwright)                                       | see `pnpm test:e2e`                                               |

## Learning experience (Phase 5)

- **Enrolment**: free courses (price 0) show **Enrol for free** on the course page and enrol instantly
  (`enrol_free`). Paid courses get checkout in Phase 6. Admins can still enrol people manually.
- **Course player** (`/learn/:slug` jumps to the next unfinished lesson): outline with completion
  ticks, progress bar, previous/next, video embeds (YouTube via youtube-nocookie, Vimeo with
  do-not-track, Mux; any other host is refused), Markdown notes, private downloads via 2-minute signed
  links, quizzes and live-session lessons. Staff can preview any course they teach.
- **Progress**: video/article/file lessons → "Mark complete"; quiz lessons → passing; live-session
  lessons → attendance recorded on a session linked to that lesson (Studio → session → "Counts
  towards lesson").
- **Quizzes** are graded inside Postgres (`submit_quiz`). Students never receive the answer key; a
  failed attempt shows only which answers were wrong, a pass reveals correct answers and explanations.
  Every attempt is stored.
- **Certificates**: issued automatically when every non-live lesson is complete (so every quiz is
  passed); attendance counts towards progress but isn't required. PDF at
  `/certificates/:serial/pdf` (holder or admin only), public check at `/verify/:serial`, emailed on issue.
- **My courses** page; the dashboard shows "Continue learning", certificates and upcoming sessions.
- The GDPR export now includes enrolments, progress, quiz attempts, registrations, certificates and
  notifications.

## Live classes (Phase 4)

- **Public**: `/schedule` (list / week / month calendar, keyboard-navigable, times in the viewer's
  timezone with a switcher), `/schedule/:id` (register, cancel, waitlist, `.ics` download and Google
  Calendar link, `Event` JSON-LD). Course pages list their upcoming sessions; the dashboard lists yours.
- **Who can register**: students with an active enrolment in the course; _open_ sessions (no course)
  accept anyone with an account. When full, people join a waitlist and are promoted automatically
  (oldest first) when someone cancels or capacity is raised. All of this runs in Postgres functions
  that lock the session row, so capacity can never be exceeded.
- **Join links** are stored separately and only returned to registered attendees from 15 minutes
  before the start until the end (staff always). They never appear in `.ics` files or emails.
- **Studio** (`/studio/sessions`): create single or weekly sessions (same local time across DST),
  edit/reschedule (attendees emailed), cancel one or the rest of a series (emailed), raise capacity
  (waitlist promoted and emailed), roster, attendance (tick boxes or CSV import with an `email`
  column, e.g. a Zoom export), CSV export, message attendees, recording URL + materials.
- **Reminders**: `GET /api/cron/reminders` sends 24-hour and 1-hour reminders; each session is
  claimed atomically so nobody gets duplicates. `vercel.json` runs it every 15 minutes and Vercel
  sends `Authorization: Bearer $CRON_SECRET`. **Vercel Hobby only allows daily cron jobs**; on Hobby,
  call the endpoint every 15 minutes from Supabase `pg_cron` + `pg_net` or any uptime pinger instead.
  Test locally:

  ```bash
  curl -H "Authorization: Bearer local-dev-cron-secret" http://localhost:3000/api/cron/reminders
  ```

- **Demo sessions** (`supabase/seeds/03_sessions.sql`): 3 weekly sessions per course starting one week
  after the seed date at 18:00 Africa/Accra (90 min, capacity 25), plus the free "Intro to Data & AI"
  session 3 days after the seed date. Join links point to `example.com`; replace them in the studio.
- Attendance is recorded now and counts towards course progress in Phase 5.

## Content management (Phase 3)

| Where                                                                                         | Who                                     | What                                                                                                                                             |
| --------------------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `/studio/courses`                                                                             | instructors (own courses), admins (all) | create, edit, publish / unpublish / archive courses; thumbnail upload                                                                            |
| `/studio/courses/:id/curriculum`                                                              | same                                    | sections and lessons: add, rename, delete, reorder by drag-and-drop (mouse, touch or keyboard) **or** Move up / Move down buttons                |
| `/studio/courses/:id/lessons/:lessonId`                                                       | same                                    | lesson type, summary, duration, free preview, move to another section, video URL, Markdown notes with preview, private file upload, quiz builder |
| `/studio/courses/:id/faqs`                                                                    | same                                    | course FAQs                                                                                                                                      |
| `/admin/users`                                                                                | admins                                  | search, filter by role, change role, suspend (blocks sign-in), manual enrol / remove access                                                      |
| `/admin/instructors`, `/admin/faqs`, `/admin/testimonials`, `/admin/settings`, `/admin/audit` | admins                                  | public instructor profiles, site FAQs, testimonials (consent required), brand/contact/**bank details**/social links, audit log                   |

**Demo content** (`supabase/seeds/02_courses.sql`, all flagged Demo): 3 published courses (Applied AI,
Data Science with Python & SQL, Power BI Mastery) with 24 sections, 86 lessons, one quiz each, course
FAQs, 4 site FAQs and 3 _placeholder_ instructors. All prices are 0 (free) for now; change them in the
course editor. The demo instructor account is linked to the Applied AI course.

**Access rules**: anyone can see published course pages and syllabus titles; lesson _content_ is
readable only for free-preview lessons, enrolled students or course staff; quiz answers are readable
only by course staff; lesson downloads live in a private bucket.

## Security model (Phase 2)

- **Roles** (`student`, `instructor`, `admin`) live in `profiles.role`. New users are always
  `student`; signup metadata is ignored for roles. A trigger blocks non-admins from changing
  role, suspension, email or the demo flag, and stops admins demoting/suspending themselves.
- **RLS** on every table; pgTAP + integration tests cover anon/student/instructor/admin access.
- **Sessions** use httpOnly, SameSite=Lax cookies (Secure in production). All Supabase calls run
  on the server; the browser never sees tokens. Pages check the user with `getUser()` (verified
  with the auth server), never by trusting the cookie alone.
- **Validation**: the same zod schemas run in the browser and again in every server action.
- **Rate limits**: sign-in 10 / 15 min, sign-up 5 / h, email links 5 / h, sensitive account
  actions 10 / h, per IP and per account, on top of Supabase Auth's limits. Relies on the
  platform setting `x-forwarded-for` (Vercel does).
- **Generic errors**: login says "Email or password is incorrect"; magic-link, reset and resend
  respond identically whether or not the email exists.
- **Passwords**: ≥ 10 chars with letters and digits (enforced by Supabase too); changing it needs
  the current password; a "password changed" email is sent.
- **Email change** needs confirmation from both old and new addresses.
- **GDPR**: "Download my data" (JSON) and "Delete account" (erases the auth user, profile and
  avatar files, then emails a confirmation) under Account → Login & security.
- **Headers**: X-Frame-Options DENY, nosniff, strict referrer policy, HSTS, Permissions-Policy.

### Production Supabase settings to mirror

`supabase/config.toml` only configures the local stack. In the hosted project dashboard set:
Site URL + redirect URLs, email confirmations on, secure email change on, "require current
password" on, minimum password length 10 with letters + digits, the four email templates from
`supabase/templates/` (paste their HTML), a custom SMTP (e.g. Resend) and sensible rate limits.

## Decisions so far

- Currency: **GHS and USD**, each price stores its own currency.
- Default timezone: **Africa/Accra** (GMT, no daylight saving). Users can choose their own.
- Google login: **deferred** (email/password and magic link only at launch).
- No Stripe at launch; provider-agnostic payments arrive in Phase 6.
