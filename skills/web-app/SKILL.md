---
name: web-app
description: How to build Julian's web apps - Bun + TypeScript + React + shadcn/ui (default styles, no custom CSS), with a single SQLite file and a tiny Bun API that also serves the built React app. Covers project layout, dev workflow, and the auth ladder (none, then local auth, then Convex Auth or Clerk). Use when creating or extending any browser-based app, dashboard, internal tool, or self-hosted service.
---

# Web app

Read `app-stack` first for the shared rules (Bun always, TypeScript always, shadcn default styles, no custom CSS). This skill is the web specifics.

## Default architecture: one process, one SQLite file

For anything single-user or self-hosted:

```
Browser  ->  Bun server (Bun.serve)  ->  SQLite file
              |- /api/*   JSON API
              |- /*       built React app (dist/), SPA fallback to index.html
```

One deployable, one port, one data file. No separate frontend host, no ORM, no queue, no cache.

### Layout

```
my-app/
  package.json            # scripts below
  bun.lock
  tsconfig.json           # strict
  components.json         # shadcn
  vite.config.ts
  index.html
  src/                    # React app
    main.tsx
    App.tsx
    components/ui/        # shadcn components (generated, do not restyle)
    lib/api.ts            # typed fetch helpers
  server/
    index.ts              # Bun.serve: API + static
    db.ts                 # bun:sqlite + migrations
  shared/
    types.ts              # types used by both src/ and server/
  Dockerfile              # from the deploy skill
  .env.example
```

### Scaffold

```bash
bun create vite my-app --template react-ts
cd my-app && bun install
bunx --bun shadcn@latest init        # accept defaults; do not customise the theme
bunx --bun shadcn@latest add button card input   # add components as needed
```

shadcn's Vite setup needs Tailwind and the `@/*` path alias in both `tsconfig` files and `vite.config.ts`. Follow the current shadcn Vite docs if `init` asks for it. Vite is the frontend build tool; **Bun is the package manager and server runtime**.

### Scripts

```json
{
  "scripts": {
    "dev": "bun run dev:server & bun run dev:web",
    "dev:web": "vite",
    "dev:server": "bun --watch server/index.ts",
    "build": "tsc --noEmit && vite build",
    "start": "NODE_ENV=production bun server/index.ts",
    "typecheck": "tsc --noEmit"
  }
}
```

In dev, Vite serves the UI and proxies `/api` to the Bun server:

```ts
// vite.config.ts (relevant part)
server: { proxy: { "/api": "http://localhost:3000" } }
```

### Server (shape to follow)

```ts
// server/index.ts
import { db } from "./db";

const port = Number(process.env.PORT ?? 3000);
const dist = new URL("../dist/", import.meta.url).pathname;

Bun.serve({
  port,
  async fetch(req) {
    const url = new URL(req.url);

    if (url.pathname === "/api/health") return Response.json({ ok: true });
    if (url.pathname.startsWith("/api/")) return handleApi(req, url);

    // static files, with SPA fallback
    const file = Bun.file(dist + url.pathname.slice(1));
    if (url.pathname !== "/" && (await file.exists())) return new Response(file);
    return new Response(Bun.file(dist + "index.html"));
  },
});
```

- Always expose `GET /api/health` (Docker healthcheck uses it).
- Read `PORT` and `DATA_DIR` from the environment.
- Validate request bodies at the API boundary (a small hand-written check or `zod` if there are many shapes). Share types via `shared/types.ts`.

### SQLite

```ts
// server/db.ts
import { Database } from "bun:sqlite";
import { mkdirSync } from "node:fs";

const dir = process.env.DATA_DIR ?? "./data";
mkdirSync(dir, { recursive: true });

export const db = new Database(`${dir}/app.db`, { create: true });
db.run("PRAGMA journal_mode = WAL");
db.run("PRAGMA foreign_keys = ON");

// Migrations: ordered list, tracked with PRAGMA user_version.
const migrations = [
  `CREATE TABLE items (id INTEGER PRIMARY KEY, title TEXT NOT NULL, created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP)`,
];
const current = db.query<{ user_version: number }, []>("PRAGMA user_version").get()!.user_version;
for (let i = current; i < migrations.length; i++) {
  db.transaction(() => {
    db.run(migrations[i]);
    db.run(`PRAGMA user_version = ${i + 1}`);
  })();
}
```

- Plain SQL with `bun:sqlite` prepared statements. No ORM unless the schema gets genuinely large.
- Only ever append to `migrations`; never edit an applied one.
- The data directory is what gets mounted as a Docker volume. Never write state anywhere else.

## UI rules
- shadcn components with default styling only. No `.css` files beyond what `shadcn init` generated (`index.css` with Tailwind and theme tokens, untouched).
- No arbitrary Tailwind values, no inline `style`, no custom colors or fonts.
- Light/dark: use what shadcn's defaults provide; do not invent a theme.
- Prefer composing `Card`, `Table`, `Dialog`, `Form`, `Input`, `Button`, `Tabs`, `Sonner` (toasts), `Skeleton`.
- Handle loading, empty, and error states for every data view.

## Auth ladder

**Step 0: none (default).** Ship the first version with no auth at all.

**Step 1: local auth (when it will be reachable by others, but is behind Tailscale).** Keep it tiny:
- One user. Password hash in an env var (`APP_PASSWORD_HASH`, produced with `Bun.password.hash`). No signup flow.
- `POST /api/login` verifies with `Bun.password.verify`, creates a random session token, stores it in a `sessions` table, sets an `HttpOnly; SameSite=Lax; Secure` cookie (drop `Secure` only for plain-http local dev).
- Middleware in `server/index.ts` rejects `/api/*` (except `/api/health` and `/api/login`) without a valid session with `401`; the UI shows a login `Card`.
- No OAuth, no email, no password reset.

**Step 2: real auth for broader use.**
- App is on Convex: **Convex Auth**. Note that Convex then replaces SQLite as the database; this is a different architecture, so confirm with Julian before migrating.
- Otherwise: **Clerk**.

Only move up the ladder when asked or when the deployment target requires it.

## Cloudflare-bound apps
If the app is small and stateless and will go to Cloudflare (see `deploy`), there is no `bun:sqlite` and no long-lived process. Write the API as a plain `fetch(req) => Response` handler (Hono is fine) so it runs on both `Bun.serve` and Workers, and keep state out of the server.

## Done checklist
- [ ] `bun install` from clean, `bun run typecheck`, `bun run build` pass
- [ ] `bun run start` serves UI and API on one port; `/api/health` returns `{ ok: true }`
- [ ] Data persists across restarts in `DATA_DIR`
- [ ] `.env.example` lists every variable; `.gitignore` covers `node_modules`, `dist`, `data`, `.env`
- [ ] Dockerfile and compose file added from `deploy` templates, and `docker build` succeeds
