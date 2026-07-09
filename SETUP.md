# SETUP.md — Claude's runbook for bootstrapping this template

This file is written **for Claude** (or any coding agent). If a human just cloned
this template and asked you to "get it running" or "set it up," follow this
runbook top to bottom. It walks the user through creating the third-party
accounts, assembling `.env.local`, and getting the dev server up.

> **Human reader?** You want the [README](./README.md) instead, or run the
> interactive `/setup-project` skill in Claude Code. This file is the agent's
> procedure, not a tutorial.

---

## How to use this runbook

1. **Read the lay of the land first:** skim `CLAUDE.md` (project conventions) and
   `README.md` (stack + scripts). Don't re-derive what they already state.
2. **Work top to bottom.** The steps are ordered by dependency — don't skip ahead.
3. **You cannot create the user's accounts for them.** For every credential, tell
   the user exactly where to go and what to click, then **ask them to paste the
   value back to you.** Never invent, guess, or use a placeholder credential.
4. **Decide the scope with the user before you start** (see next section). Not
   every service is needed to boot the app.
5. **Confirm each step succeeded before moving on** (install exits 0, migration
   runs clean, dev server logs `Ready`).

### Guardrails (do not violate)

- **Never commit `.env.local`.** It is gitignored — keep it that way. Never `git add` it.
- **Never echo full secret values back** in your chat responses or in commits.
  When confirming, show only that a variable is *set* (e.g. `TURSO_AUTH_TOKEN ✓`),
  not its contents.
- **Never paste secrets into code** or any tracked file. They live only in `.env.local`.
- Write `.env.local` with the Edit/Write tools; don't `echo secret >> .env.local`
  in a way that leaks the value into shell history or transcripts more than necessary.

---

## What the app actually needs (scope this first)

Ask the user which features they want running, because the required credentials
depend on it. The dev server will **boot with everything blank** and render the
home page — but each feature fails until its variables are set.

| Feature | Env vars required | When it's needed |
| --- | --- | --- |
| **Core auth** (sign up / in / out, sessions) | `TURSO_DATABASE_URL`, `TURSO_AUTH_TOKEN`, `SESSION_SECRET` | Almost always — this is the baseline |
| **AI chat** (`/api/chat`) | `ANTHROPIC_API_KEY` | Only if using the AI features |
| **File storage** (`src/lib/r2.ts`) | `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME` | Only if the app stores/serves files |
| **Real email delivery** | `RESEND_API_KEY`, `RESEND_FROM_EMAIL` | Production only — **in dev, leave blank and OTPs print to the server console** |

**Minimum to a working, testable app in dev:** Turso (2 vars) + `SESSION_SECRET`.
With just those, a user can sign up and sign in — the OTP code appears in the
`pnpm dev` console instead of an email. Add Anthropic and R2 later when you build
those features.

Offer the user two paths:
- **Minimal** — Turso + `SESSION_SECRET` now; stub the rest. Fastest to a running app.
- **Full** — all four services. Do this if they want AI, uploads, and email working end to end.

---

## Step 1 — Prerequisites

Check the toolchain before touching accounts:

```bash
node --version   # need >= 22.11
pnpm --version   # this repo uses pnpm; corepack enable / npm i -g pnpm if missing
```

If Node is too old, tell the user to install Node 22.11+ (nvm: `nvm install 22 && nvm use 22`).

## Step 2 — Install dependencies

```bash
pnpm install
```

Must exit cleanly. (Peer warnings about `typescript` are pre-existing and harmless.)

## Step 3 — Create `.env.local`

Check whether it already exists first — **do not clobber a user's existing secrets.**

```bash
test -f .env.local && echo "exists — do NOT overwrite" || cp .env.example .env.local
```

Then fill it in section by section using the walkthroughs below. Edit `.env.local`
in place; ask the user for each value.

---

## Step 4 — Walk the user through each account

For each service: point them to the dashboard, tell them what to create, and ask
them to paste the value. Set it in `.env.local` immediately, then move on.

### 4a — Turso (database) — *required for auth*

Turso hosts the libsql/SQLite database. The user needs a **database URL** and an
**auth token**.

Tell the user:
1. Go to **https://turso.tech** and sign up (GitHub login is easiest).
2. Create a database — either in the dashboard ("Create Database") or with the CLI:
   ```bash
   # optional CLI path
   curl -sSfL https://get.tur.so/install.sh | bash   # installs the turso CLI
   turso auth login
   turso db create ai-starter
   turso db show ai-starter --url            # -> TURSO_DATABASE_URL (libsql://...)
   turso db tokens create ai-starter         # -> TURSO_AUTH_TOKEN
   ```
3. From the dashboard: the database page shows the `libsql://…` URL, and
   "Create Token" (or the CLI command above) yields the auth token.

Set:
```
TURSO_DATABASE_URL=libsql://ai-starter-<username>.turso.io
TURSO_AUTH_TOKEN=<the token>
```

### 4b — `SESSION_SECRET` (auth) — *required for auth*

This signs the JWT session cookies. The app **throws if it is missing or shorter
than 32 characters**, so generate a strong one — don't hand-write it:

```bash
openssl rand -base64 32        # 44-char output, safely > 32
# on macOS you can pipe to the clipboard: openssl rand -base64 32 | pbcopy
```

Set:
```
SESSION_SECRET=<generated value>
```

You can run this command yourself and write the result — it's a generated secret,
not a user credential, so you don't need to ask them for it.

### 4c — Anthropic (AI) — *only for AI chat*

Powers the `/api/chat` route via the Vercel AI SDK (`@ai-sdk/anthropic`). The SDK
reads `ANTHROPIC_API_KEY` from the environment automatically.

Tell the user:
1. Go to **https://console.anthropic.com/** and sign up.
2. Add billing/credits (the API is pay-as-you-go; a key without credits will 401).
3. **API Keys → Create Key**, copy it (starts with `sk-ant-`).

Set:
```
ANTHROPIC_API_KEY=sk-ant-...
```

If the user isn't using AI yet, leave it blank and skip — the rest of the app runs fine.

### 4d — Cloudflare R2 (storage) — *only for file storage*

S3-compatible object storage. Needs the **account ID**, an **API token**
(access key + secret), and a **bucket name**.

Tell the user:
1. Go to **https://dash.cloudflare.com** and open **R2 Object Storage** in the sidebar.
2. Enable R2 if prompted (requires adding a payment method; there's a free tier).
3. **Create a bucket** — its name is `R2_BUCKET_NAME`.
4. Find the **Account ID** — it's shown in the R2 overview page (and in the
   dashboard URL). That's `R2_ACCOUNT_ID`.
5. **Manage R2 API Tokens → Create API Token**, permission **Object Read & Write**
   (scope it to the one bucket if you want least privilege). On creation it shows
   the **Access Key ID** and **Secret Access Key** once — copy both now.

Set:
```
R2_ACCOUNT_ID=<account id>
R2_ACCESS_KEY_ID=<access key id>
R2_SECRET_ACCESS_KEY=<secret access key>
R2_BUCKET_NAME=<bucket name>
```

If storage isn't used yet, leave these blank and skip.

### 4e — Resend (email) — *optional; production only*

Sends the OTP sign-in emails. **In development you do not need this** — if either
`RESEND_API_KEY` or `RESEND_FROM_EMAIL` is blank, the app prints the OTP to the
`pnpm dev` console (look for `[dev] OTP for <email>: <code>`). That's the fastest
way to test auth locally.

For real delivery, tell the user:
1. Go to **https://resend.com** and sign up.
2. **API Keys → Create API Key**, copy it (starts with `re_`).
3. **Domains → Add Domain** and add the shown DNS records to verify a sending
   domain. (For quick tests you can send from `onboarding@resend.dev`, but it only
   delivers to your own account's verified address.)
4. `RESEND_FROM_EMAIL` must be `Display Name <address@verified-domain>`.

Set:
```
RESEND_API_KEY=re_...
RESEND_FROM_EMAIL=AI Starter <noreply@yourdomain.com>
```

---

## Step 5 — Run database migrations

Applies the Drizzle migrations to the Turso DB (creates the `users` and
`otp_codes` tables):

```bash
pnpm db:migrate
```

If the database is **brand new and empty**, this succeeds cleanly. If it fails
because tables already exist (e.g. the user previously ran `pnpm db:push`), see
Troubleshooting.

## Step 6 — Start and verify

```bash
pnpm dev
```

- The server should log `✓ Ready` and serve **http://localhost:3000**.
- Have the user open it: they should see the **AI Starter** landing page with
  **Sign In** / **Sign Up** buttons and no console errors.
- Smoke-test auth (minimal path): go to **Sign Up**, enter a name + email, submit,
  then read the OTP from the `pnpm dev` console (dev/no-Resend mode), enter it, and
  confirm you land signed in.

If it works, the setup is done. Summarize which services are live and which were
left as stubs (so the user knows what to configure before deploying).

---

## Troubleshooting

- **`SESSION_SECRET must be set and at least 32 characters`** — the secret is
  missing or too short. Regenerate with `openssl rand -base64 32` and re-set it.
- **`pnpm db:migrate` fails with "table already exists"** — the DB already has the
  schema (usually from a prior `pnpm db:push`). Either point at a fresh Turso
  database, or make the generated migration idempotent by editing the SQL in
  `drizzle/` to use `CREATE TABLE IF NOT EXISTS` / `CREATE UNIQUE INDEX IF NOT EXISTS`.
- **Auth requests 500 / DB connection errors** — `TURSO_DATABASE_URL` or
  `TURSO_AUTH_TOKEN` is wrong or unset. The URL must start with `libsql://`.
- **OTP email never arrives** — expected in dev. The code is in the `pnpm dev`
  console. For real email, set both Resend vars and verify the sending domain.
- **AI chat 401 / auth error** — `ANTHROPIC_API_KEY` is missing, wrong, or the
  account has no credits.
- **Changed `.env.local` but nothing updated** — restart `pnpm dev`; env is read
  at boot.
- **`drizzle-kit` can't find env vars** — it loads `.env.local` via `dotenv` in
  `drizzle.config.ts`; make sure the file exists at the repo root.

## Related

- `README.md` — human-facing setup guide, scripts table, project structure.
- `CLAUDE.md` — project conventions you must follow when writing code here.
- `/setup-project` skill — the interactive, one-step-at-a-time version of this
  runbook for Claude Code. Prefer it when the user wants a guided flow.
- `.env.example` — the source of truth for the variable list; `.env.local` mirrors it.
