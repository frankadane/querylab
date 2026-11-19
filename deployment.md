# Deploying QueryLab

This guide takes you from the repository to a live site on **Vercel** (app) and **Supabase**
(database, auth, storage). Plan for about an hour the first time. Each step says where to click.

> Never commit secrets. Everything sensitive goes into the Vercel and Supabase dashboards.

---

## 1. Create the Supabase project

1. Go to <https://supabase.com/dashboard> → **New project**.
2. Name it `querylab`, choose a strong database password (save it in a password manager) and pick the
   region closest to your learners (for Ghana, **West EU (London)** or **Central EU (Frankfurt)**
   usually has the lowest latency).
3. Wait for the project to finish provisioning.

## 2. Push the database schema

On your computer, in the project folder:

```bash
pnpm install
npx supabase login
npx supabase link --project-ref <your-project-ref>
npx supabase db push
```

`<your-project-ref>` is the ID in your project's URL (`https://supabase.com/dashboard/project/<ref>`).
`db push` applies every file in `supabase/migrations/`, which creates all tables, row-level security
policies, functions and storage buckets.

### Demo data: keep it out of production

`supabase/seeds/01–03` are for **local development only**: they create demo accounts with published
passwords, and the demo courses and sessions depend on those accounts. Don't run them against
production. Create your real courses in **Studio → Courses** instead (you can copy the demo syllabi
from `supabase/seeds/02_courses.sql` as a starting point).

The four site pages (About, Privacy, Terms, Cookie notice) are needed in production. **After step 7**
(once your admin account exists), open **SQL Editor** in Supabase and run the contents of
`supabase/seeds/04_content.sql`. It also adds two demo blog posts that you can delete in
**Admin → Blog**.

## 3. Configure Supabase Auth

In **Authentication → URL Configuration**:

- **Site URL:** `https://YOUR-DOMAIN`
- **Redirect URLs:** `https://YOUR-DOMAIN/**` (add `https://*-YOUR-VERCEL-TEAM.vercel.app/**` if you
  want preview deployments to work)

In **Authentication → Providers → Email**:

- Enable **Confirm email**
- Enable **Secure email change**
- Enable **Require current password when updating**
- **Minimum password length:** `10`, **Password requirements:** letters and digits

In **Authentication → Emails → Templates**, paste the HTML from `supabase/templates/` into the matching
template (Confirm signup, Magic link, Reset password, Change email address) and the subjects from
`supabase/config.toml`. Enable the **Password changed** notification with
`password_changed_notification.html`.

In **Authentication → Emails → SMTP Settings**, turn on custom SMTP so auth emails come from your
domain (Supabase's built-in sender is heavily rate-limited). With Resend: host `smtp.resend.com`,
port `465`, user `resend`, password = your Resend API key.

In **Authentication → Rate Limits**, keep the defaults or tighten them; the app adds its own limits on top.

## 4. Set up Resend (email)

1. Create an account at <https://resend.com>, then **Domains → Add domain**, and add the DNS records
   it shows at your DNS provider. Wait until the domain shows **Verified**.
2. **API Keys → Create API key** (sending access). Keep it for step 5.

## 5. Deploy to Vercel

1. Push the repository to GitHub, then at <https://vercel.com/new> import it. The framework preset
   is detected as Next.js.
2. In **Settings → Environment Variables** add (Production, and Preview if you use it):

| Variable                        | Value                                                               |
| ------------------------------- | ------------------------------------------------------------------- |
| `NEXT_PUBLIC_SITE_URL`          | `https://YOUR-DOMAIN`                                               |
| `NEXT_PUBLIC_SUPABASE_URL`      | Supabase → Project Settings → API → Project URL                     |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase → Project Settings → API → `anon` public key               |
| `SUPABASE_SERVICE_ROLE_KEY`     | Supabase → Project Settings → API → `service_role` key (**secret**) |
| `RESEND_API_KEY`                | from step 4                                                         |
| `EMAIL_FROM`                    | e.g. `QueryLab <hello@YOUR-DOMAIN>` (must use the verified domain)  |
| `CRON_SECRET`                   | a long random string, e.g. the output of `openssl rand -hex 32`     |

Do **not** set `RATE_LIMIT_MULTIPLIER` in production; it exists only for the test suite.

3. Click **Deploy**.
4. **Settings → Domains:** add your domain and follow the DNS instructions.

## 6. Reminder emails (scheduled job)

`vercel.json` schedules `GET /api/cron/reminders` every 15 minutes, and Vercel sends
`Authorization: Bearer $CRON_SECRET` automatically.

- **Vercel Pro:** nothing else to do. Check **Settings → Cron Jobs** after deploying.
- **Vercel Hobby (free):** only daily cron jobs are allowed, so the 15-minute schedule is rejected.
  Remove the `crons` entry from `vercel.json` and call the endpoint from somewhere else every 15
  minutes. Option A, Supabase (free): in **Database → Extensions** enable `pg_cron` and `pg_net`,
  then run in the SQL Editor:

  ```sql
  select cron.schedule('querylab-reminders', '*/15 * * * *', $$
    select net.http_get(
      url := 'https://YOUR-DOMAIN/api/cron/reminders',
      headers := jsonb_build_object('Authorization', 'Bearer YOUR-CRON-SECRET')
    );
  $$);
  ```

  Option B: any uptime monitor that can send a custom `Authorization` header.

## 7. Create your admin account

1. Open `https://YOUR-DOMAIN/register` and create your own account; confirm the email.
2. In Supabase **SQL Editor** run (with your email):

   ```sql
   update public.profiles set role = 'admin' where email = 'you@example.com';
   ```

From then on, manage roles in **Admin → Users**.

## 8. Fill in the settings before launch

In **Admin → Settings**:

- Contact email, phone and address
- **Bank details** (shown to buyers at bank-transfer checkout)
- **Invoice details:** company name, address, tax ID (TIN) and footer, replacing the placeholders

Then:

- **Admin → Pages:** rewrite About, and have a lawyer review Privacy, Terms and the Cookie notice
  (they are templates and say so at the top).
- **Admin → Instructors:** replace the placeholder instructors with real people (or delete them).
- **Studio → Courses:** set real prices, replace placeholder lesson content and video links.
- **Studio → Live sessions:** replace the `example.com` join links with your Zoom, Meet or Teams links.

## 9. Smoke test (10 minutes)

1. Register a new account and confirm the email (checks SMTP).
2. Enrol in a free course and complete a lesson.
3. Buy a paid course with a test coupon, or by bank transfer: mark the order paid in **Admin → Orders**
   and check the invoice email.
4. Register for a live session and download the `.ics` file.
5. Run the reminder job by hand and check the response is `{"ok":true,…}`:

   ```bash
   curl -H "Authorization: Bearer YOUR-CRON-SECRET" https://YOUR-DOMAIN/api/cron/reminders
   ```

6. Check `https://YOUR-DOMAIN/sitemap.xml` and `https://YOUR-DOMAIN/robots.txt`, then submit the
   sitemap in Google Search Console.

## 10. Backups and maintenance

- Supabase Pro includes daily backups and point-in-time recovery; on the free plan, export regularly
  (**Database → Backups**).
- Schema changes: add a new file in `supabase/migrations/`, test locally with `pnpm db:reset &&
pnpm db:test`, then `npx supabase db push`.
- After a migration, regenerate types with `pnpm db:types` and commit them.
