# alpine-manager — Setup

> **For Claude Code reading this**: this file is an executable runbook. Follow
> the steps in order. Where commands are listed, run them. Where a file is
> required from outside the repo, ask the user to provide it before proceeding.

---

## 1. Prerequisites

Check that these are installed; install whatever is missing.

```bash
node --version       # expect: v20.x or newer (any LTS 20+)
npm --version        # expect: 10.x or newer
git --version        # expect: 2.30+
```

If Node is missing or too old:
- **macOS**: `brew install node@20` then add to PATH
- **Windows / Linux**: install from https://nodejs.org/ (LTS)

---

## 2. Clone the repository

```bash
cd ~/Desktop                                         # or wherever you keep code
git clone https://github.com/wenzeyang1999-maker/alpine-manager.git
cd alpine-manager
```

If `git clone` fails with a permission error, the user needs to be added as a
collaborator on the GitHub repo. Ask the user to send an invite via
https://github.com/wenzeyang1999-maker/alpine-manager/settings/access — they
should add your GitHub username and you accept the email invite.

---

## 3. Get `.env.local` from the user

The repo does NOT contain secrets. You need a file at the project root called
`.env.local` with these keys filled in:

```
RESEND_API_KEY=re_...
NOTES_TOKEN=...
NEXT_PUBLIC_NOTES_TOKEN=...
SUPABASE_URL=https://<your-project>.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...
NEXT_PUBLIC_SUPABASE_URL=https://<your-project>.supabase.co
```

**Get the actual values from the user** — they should send the `.env.local`
file via 1Password / Signal / AirDrop / encrypted channel. Do NOT request the
secrets to be pasted into chat. Put the file at `./.env.local` in the project
root (same level as `package.json`).

Verify it's gitignored (should be — already in `.gitignore`):

```bash
git check-ignore .env.local      # expect output: ".env.local"
```

### Optional env vars (have dev fallbacks, set them later if needed)

| Variable | Purpose | When to set |
|---|---|---|
| `AUTH_SESSION_SECRET` | HMAC key for analyst (`alpine_session`) cookie | Production. Dev falls back to `SUPABASE_SERVICE_ROLE_KEY` (with warning). |
| `INVESTOR_SESSION_SECRET` | HMAC key for investor (`investor_session`) cookie | Production REQUIRED — throws if missing in prod. Dev has built-in fallback. |
| `IP_HASH_SALT` | Salt for hashed-IP rate limiting | Optional — has default. |
| `RESEND_AUDIENCE_ID` | Resend mailing list ID | Only if syncing subscribers to a Resend Audience. |
| `WATERMARK_ADMIN_KEY` | Auth for `/admin/watermark` route | Only if using the admin watermark page. |

Generate a `*_SECRET`:
```bash
openssl rand -hex 32
```

---

## 4. Install dependencies

```bash
npm install
```

Takes 1–2 minutes. ~500MB of `node_modules`. If it fails on optional Playwright
browsers, that's fine — the dev server doesn't need them. Tests need them.

---

## 5. Run the dev server

The user is likely already running another Next.js project on port 3000
(`alpine-landing`). To avoid a port clash, start `alpine-manager` on a
different port:

```bash
npm run dev -- -p 3002
```

Open http://localhost:3002 — landing page should load. Common smoke tests:

| Route | Expected |
|---|---|
| `/` | Marketing landing page |
| `/whitepaper` | Public whitepaper (no login required) |
| `/login` | Investor portal sign-in |
| `/manager/landing` | Manager portal landing page |
| `/api/health` (if exists) | — |

### Demo credentials

| Account | Email | Password | Access |
|---|---|---|---|
| Demo analyst | `demo@alpinedd.com` | `demo123` | demo ODD portal |
| Demo investor | `demo@alpinedd.com` | `demo123` | investor portal at `/login` → `/reports` |
| App admin (you) | (ask user) | (ask user) | `app.alpinedd.com/admin` allowlist |

---

## 6. Repo orientation (read this before editing)

```
app/
├── api/                  # all server routes (Next.js App Router APIs)
│   ├── auth/             # analyst auth (alpine_session cookie)
│   ├── investor/         # investor portal auth
│   ├── manager/          # ⭐ manager portal (this is your area)
│   ├── subscribe/        # newsletter
│   ├── whitepaper/       # whitepaper PDF download
│   └── app-portal/       # internal analyst admin APIs
├── manager/              # ⭐ manager portal pages
├── reports/              # investor portal report reader
├── whitepaper/           # public whitepaper page
├── alpine-space/         # logged-in landing portal
└── page.tsx              # marketing landing

components/
├── investor/             # investor portal components
├── app-portal/           # analyst dashboard components
├── Hero.tsx, Navbar.tsx, Subscribe.tsx, ...   # marketing
└── FloatingSubscribe.tsx, DownloadWhitepaperModal.tsx, ...

lib/
├── app-portal/           # data modules + helpers shared by analyst + investor
├── investor/             # investor session + access control + report registry
├── auth-session.ts       # analyst session (alpine_session)
├── supabase.ts           # Supabase client (server-only)
└── constants.ts          # color palette, spacing tokens

supabase/migrations/      # SQL migrations (numbered 001..023)
docs/sample/              # sample PDFs (sources for demo reports)
public/demo-docs/         # PDFs served to users (also in Supabase Storage)
middleware.ts             # combined host routing: manager.* + app.* + apex
```

If you're working on the manager portal specifically, the relevant files are:
- `app/manager/**` — pages
- `app/api/manager/**` — API routes
- `middleware.ts` — host-based routing for `manager.alpinedd.com`

---

## 7. Dev workflow

```bash
# Run dev server
npm run dev -- -p 3002

# Type check (run before pushing — catches Next.js / TS errors)
npx tsc --noEmit

# Production build (slower, but verifies bundle works)
npm run build

# Run linter
npm run lint
```

### Local sub-domain testing (for manager portal)

The manager portal lives at `manager.alpinedd.com` in production. To test locally
under the same routing as production:

1. Add to `/etc/hosts` (requires `sudo`):
   ```
   127.0.0.1 manager.localhost
   ```
2. Open `http://manager.localhost:3002` — middleware rewrites to `/manager/*`.

If you skip this, you can still visit `/manager/landing` directly on
`localhost:3002` — same content, just bypasses the host gate.

---

## 8. Shared Supabase database — important

This repo's `.env.local` points to the **same Supabase project** as the user's
`alpine-landing` repo. That means:

- ✅ All data (reports, subscribers, investors, watermark logs) is shared
- ⚠️ Any schema change (`ALTER TABLE`, `DROP TABLE`) affects both apps
- ⚠️ Tests that write data may collide with production-like data

**Before running any migration** (`supabase/migrations/*.sql`), confirm with
the user. The migrations are idempotent (use `IF NOT EXISTS` / `ON CONFLICT`)
so re-running existing ones is safe, but new ones could break the other app.

If this becomes a problem, ask the user to spin up a separate Supabase project
for dev — it's free and isolates everything. They'd give you a new `SUPABASE_URL`
+ `SUPABASE_SERVICE_ROLE_KEY` for your `.env.local`.

---

## 9. Committing & pushing

The user's git identity is pre-configured. Confirm:

```bash
git config user.name
git config user.email
```

If empty, ask the user what name/email to use, then:

```bash
git config user.name "Your Name"
git config user.email "you@example.com"
```

Standard flow:

```bash
git checkout -b feature/manager-portal-thing      # branch for the work
# ... make edits ...
git add -A
git commit -m "feat(manager): describe what changed"
git push -u origin feature/manager-portal-thing
```

Then open a PR on GitHub against `main`. Don't push directly to `main` unless
the user explicitly says so.

---

## 10. Common deploy steps (reference only — don't run unless asked)

Production runs on a Hetzner server (not Vercel). Deploy flow when the user
asks to ship:

```bash
ssh hetzner                              # or the IP / alias the user uses
cd /path/to/alpine-manager               # ask user for actual path
git pull origin main
npm install
npm run build
sudo systemctl restart alpine-manager    # or pm2 restart
```

Migrations are applied via the Supabase SQL Editor (not automatic). Env vars
on prod live in `.env.production` on the server — different file from
`.env.local`.

---

## 11. Where to ask if stuck

- README.md in this repo
- Codebase: search for the feature name (e.g. `grep -rn "investor" lib/`)
- The user

Don't guess at secrets, DB schema, or deploy targets. Confirm first.
