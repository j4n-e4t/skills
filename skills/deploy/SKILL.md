---
name: deploy
description: How Julian ships apps - Docker image by default, run on his own server (Hetzner, behind Tailscale), or Cloudflare / Vercel for small stateless apps, or Railway for stateful apps that should not run on his own server. Includes Dockerfile, compose, and .dockerignore templates for the Bun + SQLite web app. Use when containerising, choosing where to host, or deploying a finished app.
---

# Deploy

Read `app-stack` for the decision table. Build the app per `web-app` first; this skill packages and ships it.

## Choose the target

```
Is it a desktop app?                      -> no deploy; electron-builder installer (see desktop-app)
Does it need a server process or disk
(SQLite, long-lived, websockets, jobs)?
  |- yes, personal/self-hosted            -> Docker on own server (Hetzner), reached over Tailscale
  |- yes, but not on own server           -> Railway (Docker + volume)
  '- no (stateless)
       |- small API / static / edge       -> Cloudflare
       '- fits Vercel better (Next.js,
          Clerk / Convex-backed web app)  -> Vercel
```

Default when nothing says otherwise: **Docker, own server**.

## 1. Docker (default)

Templates live in `templates/`. Copy them into the project root:

- `templates/Dockerfile` - multi-stage: build with Vite, run `server/index.ts` on Bun as a non-root user
- `templates/compose.yaml` - one service, `restart: unless-stopped`, named volume for `/data`
- `templates/.dockerignore`

Contract the app must satisfy (the `web-app` layout already does):
- Listens on `PORT` (default 3000), serves UI and API from the same process
- `GET /api/health` returns 200
- All persistent state under `DATA_DIR` (`/data` in the container)
- `bun.lock` committed; server runtime deps in `dependencies` (the image installs with `--production`)

```bash
docker build -t my-app .
docker run --rm -p 3000:3000 -v my-app-data:/data my-app
curl localhost:3000/api/health
```

Verify the image builds and health passes before declaring Docker work done.

### On the own server (Hetzner + Tailscale)
- Host runs Docker and Tailscale. Run with `docker compose up -d --build`.
- Keep the app off the public internet: bind to loopback (`127.0.0.1:3000:3000`) and expose it to the tailnet with `tailscale serve --bg 3000` (HTTPS on the tailnet name).
- That is why local auth (see `web-app`) is enough for most self-hosted apps.
- Back up by copying the `/data` volume; SQLite is in WAL mode, so use `sqlite3 app.db ".backup out.db"` for a consistent copy, not a raw file copy while running.
- Do not SSH into or modify Julian's servers unless he asks and gives the details.

## 2. Cloudflare (small, stateless)

- API as a plain `fetch` handler (Hono is fine), UI as static assets, deployed as a Worker with static assets via `wrangler`.
- Use `bunx wrangler ...`; commit `wrangler.jsonc`.
- No `bun:sqlite`, no local disk. If it needs state, stop and reconsider: either it belongs on Docker/Railway, or a tiny amount of D1/KV is acceptable. Tell Julian which you chose.
- Secrets: `bunx wrangler secret put NAME`. Never put them in `wrangler.jsonc`.
- Domains are on Cloudflare, so custom domains are a dashboard or `routes` config step. Do not change DNS unless asked.

## 3. Vercel (stateless)

- Static Vite build or Next.js, with API routes as Vercel Functions. No local disk or SQLite.
- Install with Bun (`bun install`; Vercel detects `bun.lock`, otherwise set the install command to `bun install`).
- Env vars via the Vercel dashboard or `bunx vercel env add`. Deploy with `bunx vercel` (preview) and `bunx vercel --prod` only when asked.

## 4. Railway (stateful, not on own server)

- Deploy the same Dockerfile. Railway builds it and injects `PORT`; the app already reads it.
- Attach a **volume** and mount it at `/data`, with `DATA_DIR=/data`. Without a volume, SQLite is wiped on every deploy.
- Set the healthcheck path to `/api/health`.
- Run a single replica. SQLite is single-writer.
- Set secrets (e.g. `APP_PASSWORD_HASH`) as Railway variables.

## Rules
- Never run a production deploy, push an image, or change DNS/infrastructure without Julian asking for that specific action. Building and testing locally is always fine.
- Never commit `.env`, tokens, or credentials. List every needed variable in `.env.example`.
- Pin nothing exotic: `oven/bun:1` base, current LTS tooling.
