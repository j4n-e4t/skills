---
name: desktop-app
description: How to build Julian's desktop apps - Electron with Bun as package manager and dev runtime, TypeScript, and a React + shadcn/ui renderer (default styles, no custom CSS). Covers scaffolding, process architecture, secure IPC, SQLite persistence, and packaging. Use when the project needs to be a native desktop app (filesystem access, tray, global shortcuts, offline, local-first).
---

# Desktop app (Electron)

Read `app-stack` first for the shared rules (Bun always, TypeScript always, shadcn default styles, no custom CSS). This skill is the Electron specifics. Only choose Electron when a web app cannot do the job; otherwise use `web-app`.

## Stack

| Layer | Choice |
|---|---|
| Package manager, scripts | **Bun** |
| Desktop shell | **Electron** |
| Build tooling | **electron-vite** (main, preload, renderer in one config) |
| Renderer UI | React + **shadcn/ui**, default styles, no custom CSS |
| Language | TypeScript strict, in all three processes |
| Local data | SQLite via **better-sqlite3** in the main process |
| Packaging | **electron-builder** |

### Important: Bun vs Electron's runtime
Bun is the dev tool, not the app runtime. The Electron main process runs on **Electron's embedded Node**, not Bun. So:
- Use Bun for `install`, `run`, and `bunx`. Never try to launch the main process with `bun`.
- `bun:*` APIs (`bun:sqlite`, `Bun.file`, ...) are **not available** in main or preload. Use Node APIs.
- Native modules (`better-sqlite3`) must be rebuilt against Electron's ABI. Add `"postinstall": "electron-builder install-app-deps"`.

## Scaffold

```bash
bun create @quick-start/electron my-app --template react-ts
cd my-app && bun install
bunx --bun shadcn@latest init
```

If the template command does not work under Bun, run `bunx @quick-start/create-electron`, or scaffold electron-vite manually. Then wire shadcn into the renderer:
- Renderer root is `src/renderer`. Point shadcn's `components.json` aliases and the `@/*` path alias (in `tsconfig.web.json` and `electron.vite.config.ts` under `renderer.resolve.alias`) at `src/renderer/src`.
- Add Tailwind to the renderer per the shadcn Vite docs. Keep default theme tokens untouched.

```
my-app/
  electron.vite.config.ts
  electron-builder.yml
  src/
    main/        index.ts, db.ts, ipc.ts      # Node/Electron: windows, SQLite, OS access
    preload/     index.ts                     # contextBridge: the only door between worlds
    renderer/    index.html, src/ (React + shadcn, components/ui/)
    shared/      types.ts                     # IPC contract types, imported by all three
```

## Architecture rules

**Security defaults (do not loosen):**
```ts
new BrowserWindow({
  webPreferences: {
    preload: join(__dirname, "../preload/index.js"),
    contextIsolation: true,
    nodeIntegration: false,
    sandbox: true,
  },
});
```
- The renderer is a browser page. It never imports `electron`, `fs`, or `better-sqlite3`.
- Everything privileged goes through `ipcMain.handle` in main, exposed by a small typed API in preload via `contextBridge.exposeInMainWorld("api", {...})`.
- Define the API shape once in `src/shared/types.ts` (an `Api` interface) and declare `window.api` from it. Validate IPC arguments in main; treat the renderer as untrusted.
- Never load remote URLs in the main window. Open external links with `shell.openExternal` after checking the scheme is `https:`.

**Data:**
```ts
import Database from "better-sqlite3";
import { app } from "electron";
const db = new Database(join(app.getPath("userData"), "app.db"));
db.pragma("journal_mode = WAL");
```
- DB lives in `app.getPath("userData")`. Same append-only migration approach as `web-app` (`PRAGMA user_version`).
- Small settings that do not need queries can live in a JSON file in `userData`. Do not add `electron-store` or an ORM by default.

**UI:** same rules as `web-app`. shadcn components, default theme, Tailwind layout utilities only, loading/empty/error states.

## Scripts

```json
{
  "scripts": {
    "dev": "electron-vite dev",
    "build": "tsc --noEmit && electron-vite build",
    "typecheck": "tsc --noEmit",
    "package": "bun run build && electron-builder",
    "postinstall": "electron-builder install-app-deps"
  }
}
```

## Packaging
- `electron-builder` with `electron-builder.yml`: set a real `appId` (e.g. `com.julian.<name>`) and `productName`.
- Build for the current platform by default (macOS: `dmg`). Add Windows/Linux targets only when asked.
- Code signing and notarization need Julian's Apple Developer credentials. **Do not attempt them or invent credentials.** Produce an unsigned build, and tell Julian what is needed to sign.
- Desktop apps are not "deployed". Hand over the built installer from `dist/` (or `release/`, per config).
- Auto-update is out of scope unless asked.

## Done checklist
- [ ] `bun install` from clean succeeds (native module rebuilt)
- [ ] `bun run typecheck` and `bun run build` pass
- [ ] `bun run dev` opens the window and the UI renders; an IPC round-trip works
- [ ] Data survives quitting and relaunching
- [ ] `contextIsolation: true`, `nodeIntegration: false`, no `electron`/Node imports in the renderer
- [ ] `bun run package` produces an installer
