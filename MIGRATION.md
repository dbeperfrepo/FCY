# Migrating the DBE FCY app to Supabase + GitHub Pages

## What you now have

```
dbe-fcy/
├── index.html              ← your existing app file, renamed (see step 3)
├── supabase/schema.sql     ← run this in Supabase first
├── js/
│   ├── config.js           ← paste your project URL + anon key here
│   ├── supabaseClient.js   ← creates the shared `sb` client
│   ├── auth.js             ← Supabase Auth wrapper (sign up/in/out, can(), requireCan())
│   └── store.js            ← reads/writes datasets, audit log, templates, settings, invite codes
└── css/styles.css          ← (optional) pull the <style> block out here
```

## Step 1 — Create the Supabase project
1. Go to supabase.com → New project. Pick any name/region, set a database password (save it somewhere — you won't need it day-to-day, but you'll want it if you ever connect a SQL client directly).
2. Wait for provisioning (~2 min).

## Step 2 — Run the schema
1. In the Supabase dashboard: **SQL Editor → New query**.
2. Paste the entire contents of `supabase/schema.sql` and click **Run**.
3. This creates all 6 tables, the RLS policies, the invite-code signup trigger, and one **bootstrap Administrator invite code**: `BOOTSTRAP-ADMIN-0001` (valid 7 days, single use).
4. In **Project Settings → Authentication → Providers**, confirm Email is enabled. Under **Email templates / URL Configuration**, if you don't want email confirmation required before first login, you can disable "Confirm email" for now (Authentication → Providers → Email → toggle "Confirm email") — easy to re-enable later.

## Step 3 — Wire the frontend
1. Rename your current file to `index.html` in this folder.
2. Right before your existing `<script>` block, add:
   ```html
   <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js"></script>
   <script src="js/config.js"></script>
   <script src="js/supabaseClient.js"></script>
   <script src="js/auth.js"></script>
   <script src="js/store.js"></script>
   ```
3. In `js/config.js`, replace the two placeholders with your project's **Project URL** and **anon public key** (Project Settings → API).
4. In your existing script, **delete** these old blocks (they're fully replaced):
   - `lsGet`/`lsSet`/`lsDel` calls for `LS_DATA`, `LS_LOG`, `LS_SET`, `LS_TPL`, `LS_INVITE_CODES`, `LS_USERS`, `LS_SESS`
   - `hashPassword`, `setSession`, `getSession`, `clearSession`, `currentAuthUser`
   - the body of `can()` / `requireCan()` — replace calls to them with `Auth.can(...)` / `Auth.requireCan(...)`
   - `persist()` → replace call sites with `await Store.persistYear(state.year, state, rawOf)`
   - `saveLog()` / `addAudit()` → replace with `Store.addAudit(action, detail)`
   - `saveTemplates()` → replace with `await Store.saveTemplate(name, cfg)` / `await Store.deleteTemplate(id)`
   - `loadSignupCodes`/`saveSignupCodes` → replace with `await Store.loadSignupCodes()` / `await Store.createSignupCode(...)` / `await Store.revokeSignupCode(id)`
5. At the very top of your boot sequence (wherever the app currently calls `loadData(); recalcAll(); render();` on startup), change it to:
   ```js
   async function boot(){
     await Auth.restoreSession();
     if (!Auth.currentUser()) { showLoginScreen(); return; }
     await Store.loadAll(YEARS, state, rawOf);
     recalcAll();
     render();
   }
   boot();
   ```
6. Your login/signup form handlers become:
   ```js
   async function doLogin(email, password){
     try { await Auth.signIn({email, password}); await Store.loadAll(YEARS, state, rawOf); recalcAll(); render(); }
     catch(e){ showToast(e.message, 'error'); }
   }
   async function doSignup(name, email, password, inviteCode){
     try { await Auth.signUp({name, email, password, inviteCode}); showToast('Account created — check your email if confirmation is required.', 'success'); }
     catch(e){ showToast(e.message, 'error'); }
   }
   ```

This keeps ~95% of your existing rendering/report/pivot/export code untouched — only the persistence and auth call sites change.

## Step 4 — Create your first admin account
1. Open the app (locally is fine: just open `index.html`, or serve the folder with any static server).
2. Sign up with your own email/password and invite code `BOOTSTRAP-ADMIN-0001`.
3. Once logged in, delete the bootstrap code (Admin panel → Invitation Codes, or in SQL Editor: `delete from signup_codes where code='BOOTSTRAP-ADMIN-0001';`).
4. From now on, issue invite codes for new users from the Admin panel (unchanged — that UI already calls `Store.createSignupCode`).

## Step 5 — Push to GitHub and enable Pages
```bash
cd dbe-fcy
git init
git add .
git commit -m "Initial commit: DBE FCY app on Supabase"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```
Then in the GitHub repo: **Settings → Pages → Source: Deploy from a branch → Branch: main / (root)**. GitHub gives you a URL like `https://<your-username>.github.io/<repo-name>/` within a minute or two — that's the link every user opens, from any device.

## A couple of things worth knowing
- **The anon key is meant to be public.** Access control lives entirely in the RLS policies in `schema.sql`, not in hiding that key — this is the standard Supabase pattern for a frontend-only app with no separate server.
- **Login is by email now**, not username, since that's what Supabase Auth requires. `username` is still stored on the profile for display.
- **Multiple users editing at once**: this migration gives you shared, centralized storage reachable from anywhere — it does *not* add live real-time sync (e.g., someone else's edit appearing on your screen without a refresh). If you want that, it's a small addition (`sb.channel(...).on('postgres_changes', ...)` in `store.js`) — say the word and I'll add it.
- **Custom domain**: optional, configurable later under Pages settings if you want something nicer than the github.io URL.
