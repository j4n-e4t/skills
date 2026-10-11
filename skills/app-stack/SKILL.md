---
name: app-stack
description: Julian's default toolchain and decision table for building any new app (web, desktop, small tool, side project). Use FIRST whenever starting a new project or asked "what should I build this with / where should this run". Picks the platform and deploy target, then points to web-app, desktop-app, or deploy.
---

# App stack: defaults and decisions

Julian builds quick, fun projects. Optimise for **minimal, boring, fast to ship**. Do not ask about stack choices that are answered here; just use them.

## Non-negotiable for every project

| Concern | Choice |
|---|---|
| Package manager + dev runtime | **Bun**, always. Never npm, yarn, pnpm, or Node-only scripts. Use `bun`, `bunx`, `bun run`. |
| Language | **TypeScript**, always, `strict: true`. No plain `.js` source files. |
| UI | **React + shadcn/ui** |
| Styling | shadcn **default styles only**. No custom CSS (see below). |

### The no-custom-CSS rule
- Use shadcn components as installed (`bunx --bun shadcn@latest add <component>`), with the default theme `init` generates.
- Do not write CSS rules, add CSS files, change theme tokens/colors/radius/fonts, or use arbitrary values like `w-[137px]`.
- Layout via standard Tailwind utility classes (`flex`, `grid`, `gap-4`, `p-6`, `max-w-2xl`, `mx-auto`) is fine. That is composition, not custom CSS.
- If a needed component does not exist in shadcn, compose existing ones. Do not hand-style a replacement.

## Pick the platform

| The project is... | Use | Skill |
|---|---|---|
| Something used in a browser | Web app | `web-app` |
| Needs to be a real desktop app (native windows, filesystem, tray, global shortcuts, runs offline on the machine) | Electron | `desktop-app` |

If unsure, build the web app. Reach for Electron only when the browser cannot do the job.

## Pick the data layer

| Situation | Use |
|---|---|
| Single user, self-hosted, simple | **One SQLite file** behind a tiny API that also serves the React app |
| Desktop app needing persistence | SQLite in the user-data dir (main process) |
| Multi-user / realtime / broadly public | Ask Julian. Convex is the likely answer (comes with Convex Auth) |
| Stateless | No database |

## Pick auth (default: none)

1. **First run: no auth.** Do not add it until asked or until the app is going somewhere that needs it.
2. **Behind Tailscale (self-hosted, personal):** minimal local auth. Details in `web-app`.
3. **Broader use:** Convex Auth if the app is on Convex, otherwise Clerk.

## Pick the deploy target

Most apps ship as a **Docker image**. See `deploy` for templates.

| Situation | Target |
|---|---|
| Self-hosted, stateful (SQLite), personal, behind Tailscale | **Docker on Julian's own server** (Hetzner) |
| Small and **stateless** | **Cloudflare** (Workers / static assets) |
| Stateless web app that suits Vercel better (e.g. Next.js, Clerk, Convex-backed) | **Vercel** |
| Needs a stateful server, but should not run on own server | **Railway** (Docker + volume) |
| Desktop app | Packaged installer via electron-builder, not deployed |

DNS lives on Cloudflare and servers on Hetzner. This rarely matters for building; do not configure DNS unless asked.

## Working agreements
- Scaffold first, get it running end to end, then add features.
- Keep the repo small: no extra frameworks, state libraries, ORMs, or UI kits unless the project clearly needs them.
- Before saying "done": `bunx tsc --noEmit` passes, `bun run build` passes, and the app was actually started and exercised.
- Never commit secrets. Configuration comes from environment variables; commit a `.env.example`.
