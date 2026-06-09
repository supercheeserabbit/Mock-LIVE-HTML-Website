# Launch Checklist — Moving the LIVE Project Site to Reclaim Hosting

How the pieces fit together:

- **GitHub** = where the code lives (source of truth). You push code here.
- **Reclaim Hosting (cPanel)** = where the live site is served to visitors at your domain.
- **Quarto** = builds your `.qmd` pages into the `docs/` folder. cPanel does NOT run Quarto — it only copies the already-built `docs/` folder. So you always render locally first.
- **Resend** = sends the invite emails. You'll create a NEW official account.
- **Supabase** = handles login / set-password / gated reports. You'll create a NEW official project (the current one is your personal/mock project).

The domain is already connected to Reclaim and HTTPS is already active, so those steps are not in this list. Order matters: do the phases top to bottom — except Phase 1, which you should kick off immediately and let run in the background.

---

## Phase 1 — Request cPanel access (start this first, it waits on your supervisor)

Do this before anything else, because it depends on someone else responding and it unblocks both deployment (Phase 6) and the DNS records (Phase 7) — and DNS can take 24–48h to propagate, so the sooner the better. You can work through the other phases while you wait. You don't need a new hosting account, and you should NOT share your supervisor's main cPanel password.

1. Ask your supervisor to create you a **Team User** in cPanel (Manage Team → Create Team User).
2. Request these privileges explicitly: the **Web** role (covers Git Version Control + File Manager) plus **Git** and **Zone Editor** access.
   - If Zone Editor can't be granted to a team user, that's fine — he can paste in the few DNS records for you (see Phase 7).

---

## Phase 2 — Move the code to the project's GitHub org

1. In RStudio, open the **Terminal** tab (Tools → Terminal → New Terminal — this is the shell, NOT the Console, which runs R). Confirm you're in the project folder:
   ```bash
   pwd        # should show your project path
   git status # should list your files, not an error
   ```
2. Add the org repo as a remote and push your code:
   ```bash
   git remote add live https://github.com/theliveproject/theliveproject.github.io.git
   git remote -v            # confirm "origin" AND "live" are both listed
   git push live main       # use your real branch; if Git complains the repo isn't empty, see note
   ```
   - Note: if the org repo already has placeholder content, either merge once, or force-push if it's just an empty starter: `git push live main --force`.
3. From now on, the **theliveproject** repo is your source of truth.

---

## Phase 3 — Create the LIVE Project Resend account and verify the domain

Set up a fresh Resend account owned by the LIVE project (not personal), so the next phase can wire its API key into Supabase. Do the domain verification steps as early as possible — DNS propagation can take 24–48 hours.

1. **Create the official account** at [resend.com](https://resend.com), ideally using a LIVE-project email (or one a supervisor controls) so it's not tied to your personal account. The free tier (3,000 emails/month) is plenty for invites.
2. **Create an API key:** Resend → **API Keys → Create API Key**, and copy it somewhere safe — you'll paste it into Supabase in Phase 4.
3. **Create a subdomain for sending email** (e.g., `mail.YOURDOMAIN.com`). This isolates your transactional email reputation from your main domain and avoids DNS conflicts if you ever add another email provider. You or your supervisor can create it in cPanel → **Domains → Create a New Domain** (or **Subdomains**), then use `mail.YOURDOMAIN.com` as the domain in the next step.
4. In Resend → **Domains → Add Domain**, enter `mail.YOURDOMAIN.com` (your new subdomain). Resend gives you a set of TXT/CNAME/DKIM records.
5. In cPanel → **Domains → Zone Editor → Manage** (your domain) → **Add Record**, paste each record Resend provided. Save.
6. DNS changes can take up to 24–48 hours to propagate. Once Resend shows the domain as verified, the SMTP settings from Phase 4 step 4 will send successfully.

---

## Phase 4 — Create the official Supabase project

Your current Supabase project is personal/mock. Create a fresh one owned by the LIVE project so it's official and not tied to your personal account.

1. **Create the project.** At [supabase.com](https://supabase.com), ideally under a LIVE-project organization/account (or one a supervisor owns), click **New Project**. Pick a name (e.g. `live-project`), set a strong database password, and choose a region close to your users.
2. **Recreate the database schema.** In **SQL Editor**, run the `profiles` table setup from your existing `ACCOUNT_SETUP_INSTRUCTIONS.md` (the `create table public.profiles ...` block and its row-level-security policy).
3. **Set the auth URLs.** In **Authentication → URL Configuration**:
   - **Site URL:** `https://YOURDOMAIN.com/`
   - **Redirect URLs:** add `https://YOURDOMAIN.com/setpassword.html`
4. **Configure email (SMTP via Resend).** In **Authentication → SMTP Settings**, enable custom SMTP: host `smtp.resend.com`, port `465`, username `resend`, password = the Resend API key from Phase 3, sender `noreply@mail.YOURDOMAIN.com`. (Emails will actually send once the Resend domain is verified in Phase 7.)
5. **Copy the new credentials.** Go to **Settings → API** and copy the new **Project URL** and the **publishable/anon key**. You'll paste these into the code in Phase 5.
6. **Note on data:** since the old project was mock, you're starting users fresh — you'll re-invite real users in Phase 8. (If you ever need to carry data over, export/import via the SQL Editor.)

---

## Phase 5 — Prepare the code (new domain + new Supabase project)

1. **Swap the domain URLs.** Search the project for `https://supercheeserabbit.github.io/Mock-LIVE-HTML-Website/` and change to `https://YOURDOMAIN.com/` in:
   - `_navbar.html` (the logout redirect, ~line 44)
   - `login.html` (~lines 164 and 189)
   - `setpassword.html` (`REPORTS_URL`, ~line 211)
2. **Swap the Supabase credentials.** Replace the OLD project URL `https://plsrkfhxfzwzvrtciios.supabase.co` and the OLD anon key `sb_publishable_2spGzG3-1-xfAGai7r_SSQ_h3TkSHXJ` with the NEW Project URL and key from Phase 4, in all four files:
   - `_navbar.html` (~line 32)
   - `reports.qmd` (~line 11)
   - `login.html` (`SUPABASE_URL` / `SUPABASE_ANON`, ~lines 156–157)
   - `setpassword.html` (`SUPABASE_URL` / `SUPABASE_ANON`, ~lines 208–209)
3. **Add the deploy config.** Create a file named `.cpanel.yml` in the repo root so Reclaim copies the built site into your web root. Replace `USERNAME` with the cPanel username:
   ```yaml
   ---
   deployment:
     tasks:
       - export DEPLOYPATH=/home/USERNAME/public_html/
       - /bin/cp -R docs/* $DEPLOYPATH
   ```
4. **Render, commit, and push:**
   ```bash
   quarto render
   git add -A
   git commit -m "Point site at real domain and official Supabase project; add cPanel deploy config"
   git push live main
   ```

---

## Phase 6 — Set up the Git deployment in cPanel (this replaces the old code)

Requires your cPanel access from Phase 1.

1. cPanel → **Files → Git Version Control → Create**.
2. Clone URL: the **theliveproject** repo.
   - Public repo: use the HTTPS URL (`https://github.com/theliveproject/theliveproject.github.io.git`).
   - Private repo: generate an SSH key in cPanel (SSH Access), add it to GitHub as a deploy key, and use the SSH URL (`git@github.com:...`).
3. Clone path: a folder OUTSIDE `public_html` (e.g. `repositories/live-site`). The `.cpanel.yml` handles copying into `public_html`.
4. Deploy: open the repo's **Manage → Pull or Deploy → Update from Remote**, then **Deploy HEAD Commit**. This overwrites the old code currently in `public_html` with yours.

---

## Phase 7 — Invite real users

In the new Supabase project: **Authentication → Users → Invite User**, enter each real user's email. They'll get the invite email (via Resend) → click the link → land on `setpassword.html` on your domain → set a password → get redirected to `reports.html`. (Optional: add their name/institution rows to the `profiles` table per `ACCOUNT_SETUP_INSTRUCTIONS.md`.)

---

## Phase 8 — Verify everything

1. Visit `https://YOURDOMAIN.com` and confirm the padlock (HTTPS) shows.
2. Click every navbar link — Home, Overview, Our Team, Reports, Login.
3. Test the full auth loop against the NEW Supabase project: invite a test user → set-password link lands on your domain → redirects to `reports.html` → log out works.

---

## Your routine after launch (every future change)

1. Edit your `.qmd` files in RStudio.
2. `quarto render` (rebuilds `docs/`).
3. `git add -A && git commit -m "..."` then `git push live main`.
4. In cPanel → Git Version Control → **Update from Remote** → **Deploy HEAD Commit**.

(Optional later: a GitHub Action can do step 4 automatically on every push, so you skip the cPanel clicks.)

---

## One thing to know about security

The Reports page is gated with client-side JavaScript, which means a determined visitor could still download the page by bypassing the script. If the reports contain anything truly sensitive, the real protection needs to be server-side — for example cPanel directory password protection, or serving the data through Supabase with row-level security — rather than a JavaScript redirect.
