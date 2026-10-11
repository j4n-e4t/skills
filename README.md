# skills

Skills for AI agents that encode how I build and ship apps: toolchain, defaults, and deploy targets.

## Skills

| Skill | Use it for |
|---|---|
| [`app-stack`](skills/app-stack/SKILL.md) | Start here. Shared defaults and the decision table (platform, data, auth, deploy). |
| [`web-app`](skills/web-app/SKILL.md) | Bun + TypeScript + React + shadcn/ui, single SQLite file, tiny Bun API serving the UI. |
| [`desktop-app`](skills/desktop-app/SKILL.md) | Electron + Bun tooling + React + shadcn/ui. |
| [`deploy`](skills/deploy/SKILL.md) | Docker (own server), Cloudflare, Vercel, Railway. Includes Dockerfile/compose templates. |

## The short version

- **Always:** Bun, TypeScript, React, shadcn/ui with default styles and no custom CSS.
- **Web, self-hosted:** one Bun process serving the React app and a JSON API, backed by one SQLite file. No auth at first, then local auth behind Tailscale, then Convex Auth or Clerk.
- **Desktop:** Electron.
- **Ship:** Docker on my own server by default; Cloudflare or Vercel if stateless; Railway if it needs a stateful server that should not live on mine.

## Using these skills

Each skill is a folder containing a `SKILL.md`. Symlink or copy the ones you want into an agent's skills directory, for example:

```bash
ln -s "$PWD/skills/app-stack" ~/.claude/skills/app-stack
```

Repeat for `web-app`, `desktop-app`, and `deploy`.
