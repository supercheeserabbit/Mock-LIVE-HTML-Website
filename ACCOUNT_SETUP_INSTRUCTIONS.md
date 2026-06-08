# Account Creation Setup Instructions

Admins create user accounts directly in the Supabase dashboard and send invite links from there. No custom admin page or Edge Function is needed.

---

## 1. Invite a user (admin step)

Go to **Supabase Dashboard → Authentication → Users → Invite User**, enter the user's email, and click **Send Invite**. Supabase will email them a link automatically.

If you also want to store first name, last name, and institution, you can manually insert a row into the `profiles` table (see step 2) after inviting them.

---

## 2. (Optional) Create a `profiles` table

If you want to store additional user info (name, institution), run this in **Supabase Dashboard → SQL Editor**:

```sql
create table public.profiles (
  id          uuid primary key references auth.users(id) on delete cascade,
  first_name  text not null,
  last_name   text not null,
  institution text not null,
  email       text not null,
  created_at  timestamptz default now()
);

alter table public.profiles enable row level security;

create policy "Users can read own profile"
  on public.profiles for select
  using (auth.uid() = id);
```

After inviting a user, go to **Table Editor → profiles** and insert their details manually.

---

## 3. Add the invite redirect URL to Supabase

Go to **Supabase Dashboard → Authentication → URL Configuration → Redirect URLs** and add:

```
https://supercheeserabbit.github.io/Mock-LIVE-HTML-Website/setpassword.html
```

Also set **Site URL** to:
```
https://supercheeserabbit.github.io/Mock-LIVE-HTML-Website/
```

This ensures the invite email link lands on `setpassword.html` instead of Supabase's default page.

---

## 4. Configure email via Mailtrap

Supabase's built-in email is limited to ~2 invites/hour. Use Mailtrap's **Email Sending** service (not the sandbox/testing inbox) to remove that limit.

> **Important:** Mailtrap has two products — *Email Testing* (catches emails in a sandbox) and *Email Sending* (delivers to real inboxes). You need **Email Sending** for users to actually receive invites.

### Steps

1. Sign up at [mailtrap.io](https://mailtrap.io) and go to **Email Sending → Domains**
2. Add and verify your sending domain (e.g., `yourdomain.com`)
3. Go to **Email Sending → SMTP/API Settings** and copy your SMTP credentials
4. In **Supabase Dashboard → Authentication → SMTP Settings**:
   - Toggle **Enable Custom SMTP** on
   - **Host:** `live.smtp.mailtrap.io`
   - **Port:** `587`
   - **Username:** `api`
   - **Password:** your Mailtrap API token
   - **Sender email:** a verified address from your Mailtrap domain (e.g., `noreply@yourdomain.com`)
   - **Sender name:** whatever you want users to see (e.g., `Work as Calling Research`)
5. Save and send a test invite to confirm delivery

---

## 5. Verify the flow

1. Admin goes to **Supabase Dashboard → Authentication → Users → Invite User** and enters the user's email
2. User receives the invite email → clicks the link → lands on `setpassword.html`
3. User sets their password → automatically redirected to `reports.html`
4. Check **Authentication → Users** to confirm the account is active

---

## Files

| File | Purpose |
|------|---------|
| `setpassword.html` | User-facing page for setting password via invite link |
