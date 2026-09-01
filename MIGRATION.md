# DBE FCY app — Supabase + GitHub Pages setup

## What you now have

```
dbe-fcy/
├── index.html                          ← everything: your app + Supabase config/client/auth/store, all inline
└── supabase/
    ├── schema.sql                      ← run this in Supabase first
    └── functions/
        ├── _shared/admin-auth.ts       ← shared helper (not deployed on its own)
        ├── admin-create-user/index.ts  ← Edge Function: create an account
        ├── admin-delete-user/index.ts  ← Edge Function: delete an account
        └── admin-set-password/index.ts ← Edge Function: set someone's password
```

**Account model**: there is no self sign-up anywhere in this app. Every account — including the very first one — is created by an Administrator. Login is just email + password, nothing else. The three Edge Functions are what make "an Administrator creates every account" possible *securely* from a site with no backend server of its own — see the box at the end of this file for why they're necessary.

Near the top of the main `<script>` block in `index.html`:
- `window.SUPABASE_URL = '...'` and `window.SUPABASE_ANON_KEY = '...'` — **edit these** with your project's values (Step 3 below).
- The `const sb = ...`, `const Auth = ...`, and `const Store = ...` blocks right after — the data/auth layer. Nothing to edit here.

## Step 1 — Create the Supabase project
1. Go to supabase.com → **New project**. Pick any name/region, set a database password (save it — you'll want it if you ever connect a SQL client directly).
2. Wait for provisioning (~2 min).

## Step 2 — Run the schema
1. In the Supabase dashboard: **SQL Editor → New query**.
2. Paste the entire contents of `supabase/schema.sql` and click **Run**. Safe to re-run any time.
3. This creates the 5 tables (`profiles`, `datasets`, `app_settings`, `audit_log`, `templates`), all the RLS policies, and the "last Administrator can't be removed" safeguard. It does **not** create any accounts — that's Step 4.

## Step 3 — Turn off public sign-up (important)
Removing the "Create an account" button from the page isn't enough on its own — someone could still call Supabase's sign-up API directly from the browser console with your public anon key unless the project itself refuses it.
1. **Authentication → Providers → Email**.
2. Toggle **off** "Allow new users to sign up" (may be labelled "Enable Signups" depending on your dashboard version).
3. Save.

This only blocks the public `signUp()` API — it does *not* block the Edge Functions below, which use the Admin API instead. That's exactly the split we want: nobody self-registers, but an Administrator still can create accounts from inside the app.

## Step 4 — Deploy the three Edge Functions
These need the Supabase CLI (they can't be pasted into the SQL Editor — they're server-side TypeScript, not SQL).

1. Install the CLI if you don't have it:
   ```bash
   npm install -g supabase
   ```
2. From inside the `dbe-fcy` folder, log in and link your project (find your project ref in the dashboard URL: `supabase.com/dashboard/project/<project-ref>`):
   ```bash
   cd dbe-fcy
   supabase login
   supabase link --project-ref <your-project-ref>
   ```
3. Deploy all three functions:
   ```bash
   supabase functions deploy admin-create-user
   supabase functions deploy admin-delete-user
   supabase functions deploy admin-set-password
   ```
   No secrets to configure — Supabase automatically provides `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` inside every Edge Function's environment. You never see or paste that key anywhere.
4. Each `supabase functions deploy` command prints a URL — you don't need to copy it anywhere; `index.html` calls these by name (`sb.functions.invoke('admin-create-user', ...)`), which resolves automatically against your linked project.

## Step 5 — Wire the frontend
Open `index.html` and find `window.SUPABASE_URL = 'https://YOUR-PROJECT-REF.supabase.co'` (a few lines below the opening `<script>` tag). Replace it and the line below it with your project's **Project URL** and **anon public key** (Project Settings → API).

## Step 6 — Create your one Administrator account
Since there's no sign-up form, you create this one directly in the Supabase dashboard — it takes under a minute and needs no code:

1. **Authentication → Users → Add user → Create new user.**
2. Enter your own email and a password. Tick **Auto Confirm User** (so you don't need email confirmation set up yet).
3. Click **Create user**. Copy the **User UID** shown in the new row (looks like `a1b2c3d4-...`).
4. Back in **SQL Editor → New query**, run (replacing both placeholders):
   ```sql
   insert into public.profiles (id, name, username, role, privileges, active)
   values (
     '<paste the User UID here>',
     'Your Full Name',
     'youradminusername',
     'Administrator',
     public.default_privileges('Administrator'),
     true
   );
   ```
5. That's it — open the app and sign in with that email/password. Every other account from now on gets created **from inside the app**, in the Admin panel, by you.

## Step 7 — Push to GitHub and enable Pages
```bash
git init
git add .
git commit -m "Initial commit: DBE FCY app on Supabase"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```
Then in the GitHub repo: **Settings → Pages → Source: Deploy from a branch → Branch: main / (root)**. GitHub gives you a URL like `https://<your-username>.github.io/<repo-name>/` within a minute or two — that's the link every user opens, from any device.

## Day-to-day: creating, managing, and removing users
All from **Admin panel → User Management**, once you're signed in as Administrator:
- **Create Account**: name, email, role, and an initial password (type your own, or click 🎲 to generate a strong one) — then share that password with the person yourself, however you'd normally share something sensitive (not email, ideally). They can change it any time via **Forgot password?** on the sign-in screen.
- **Set Password**: reset anyone's password directly, on the spot — no email round-trip needed.
- **Privileges**: fine-tune an individual user's permissions beyond their role's defaults.
- **Deactivate / Activate**: instantly blocks or restores access without deleting the account.
- **Delete**: permanently removes the account and login. Cannot be undone.
- The database itself refuses to let the **last active Administrator** be demoted, deactivated, or deleted — even by direct API call — so you can't accidentally lock yourself out.

Every one of these actions is written to the Audit Trail automatically.

---

### Why Edge Functions, and why this couldn't just be "add a button"
Creating another person's login, or setting their password directly, requires Supabase's **service-role key** — a credential that bypasses every access rule in the database. If that key were embedded anywhere in `index.html`, anyone visiting your GitHub Pages site could open dev tools, read it out of the page source, and get unrestricted access to every table — completely defeating the Row Level Security this whole setup depends on. An Edge Function is Supabase's own server-side code — the service-role key lives only inside it, is never sent to any browser, and each function independently re-confirms (against your real database, not anything the browser claims) that whoever's calling it is genuinely a signed-in, active Administrator before it does anything.

### A couple of other things worth knowing
- **The anon key is meant to be public.** It has no power on its own — access control lives entirely in the RLS policies in `schema.sql` and the admin checks inside each Edge Function.
- **Login is by email**, not username — that's what Supabase Auth requires. `username` is still stored on the profile for display.
- **Multiple users editing at once**: this setup gives everyone shared, centralized storage reachable from anywhere — it does *not* add live real-time sync (someone else's edit appearing on your screen without a refresh). If you want that later, it's a small addition — just ask.
- **Custom domain**: optional, configurable later under Pages settings if you want something nicer than the github.io URL.
