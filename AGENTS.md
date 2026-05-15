# AGENTS.md — FM-broadcaster

Project-level conventions and context for AI agents (Devin, Copilot, etc.) working in this repository.

---

## What this project is

A **zero-dependency micro-app** (`index.html` + sibling `onfmready.js`) designed to be loaded
inside FileMaker Pro **Web Viewer** panels — one instance per layout/window. Its purpose is
to give every simultaneously-open WebViewer a shared, real-time **key/value store** that any
instance can create, read, update, or delete. Mutations are reflected instantly across all
open instances, persisted to `localStorage` for durability, and reported to FileMaker via
`FileMaker.PerformScript`.

Scope: same FileMaker Pro client process (BroadcastChannel + localStorage are origin-scoped).
Multiple `.fmp12` files in the same FM client can share the store if they all point
WebViewers at the same hosted URL.

No server. No build step. No npm.

---

## Repository layout

```
index.html         — The active, production-ready implementation (HTML/CSS/JS)
onfmready.js       — External FileMaker bridge helper (sibling file, MIT, must be served alongside)
FM-Broadcaster.fmp12 — Companion FileMaker file with example handler script
.gitignore         — Excludes old/ and macOS artifacts
AGENTS.md          — This file
README.md          — Full developer + integrator documentation
old/               — Local-only archived prototypes (gitignored; do not touch)
```

`old/` is gitignored and should never be modified, referenced, or used as a source of truth.

---

## Source file: index.html — logical sections

| Block | Purpose |
|-------|---------|
| `<script src="onfmready.js">` | Sibling-file include of onfmready.js v2.1.11 (MIT). **Must remain the first `<script>` in the document.** Queues `FileMaker.PerformScript()` calls until the FM object is injected. Deploy `onfmready.js` next to `index.html` at the same URL path. |
| `<style>` | Self-contained UI styles. No external CSS. Uses plain values (no CSS variables) so it works standalone. |
| `<body>` markup | Static shell: header badge, peers row, add-row inputs, store table div, edit modal. |
| `<script>` runtime | All application logic: identity, CONFIG (URL params), state, persistence, channel, FM bridge, UI. |

---

## JavaScript architecture

### Constants / identity

```
INSTANCE_ID     — crypto.randomUUID(), unique per WebViewer instance
INSTANCE_COLOR  — one of 7 preset hex colours, randomly picked
SHORT_ID        — last 6 chars of INSTANCE_ID, shown in badge
```

### CONFIG block (URL-overridable)

Resolved once at startup via `paramStr` / `paramBool` from `URLSearchParams`. Defaults are
hard-coded in the `CONFIG` block; URL query parameters override them per-WebViewer.

| Constant | URL param | Default |
|----------|-----------|---------|
| `CHANNEL_NAME` | `channel`     | `fm-broadcaster` |
| `STORAGE_KEY`  | `storageKey`  | `fm-broadcaster:state` |
| `FM_SCRIPT`    | `fmScript`    | `fm-broadcaster.event_handler` |
| `PERSIST`      | `persist`     | `true` |

All four are `const` after resolution. To add a new knob, place it inside the `CONFIG`
block (do not scatter URL-param reads through the codebase).

### State

```
store     — { [key: string]: string }       in-memory key/value store
storeMeta — { [key: string]: { ts, origin, deleted } }  per-key last-writer-wins meta
peers     — { [instanceId: string]: color } known live peer instances
```

### Key functions

| Function | Description |
|----------|-------------|
| `applyMutation(cmd, key, value, token, broadcast, meta)` | **Single source of truth** for all store mutations. Updates `store` + `storeMeta`, calls `render()` and `persist()`, optionally broadcasts. Returns `{ oldValue, newValue, changed }`. When `meta` is supplied (incoming peer mutation) it is honoured for last-writer-wins; otherwise a fresh `nextMeta()` is generated. |
| `mergeFullState(data)` | Applies an incoming `sync_full` snapshot. **Fast path**: if local `store` is empty, replaces it wholesale and persists. Otherwise per-key merge via `isNewerMeta`. |
| `loadPersisted()` / `persist()` | localStorage IO under `STORAGE_KEY`. No-ops when `PERSIST=false`. |
| `callFM(cmd, data, token)` | Central choke-point for all outbound `FileMaker.PerformScript` calls. Builds the standard `{cmd, data, token}` payload. |
| `notifyFmLoaded()` | Sends `cmd: "loaded"` to FM with current `store` snapshot. |
| `notifyFmChange(key, old, new, token)` | Sends `cmd: "onChange"` to FM with change details. |
| `window.fmCommand(payloadStr)` | **Entry point called by FileMaker** via `Perform JavaScript in Web Viewer`. Parses payload, dispatches to `applyMutation`. |
| `requestSync()` | Broadcasts `request_sync` repeatedly on load (late-join protocol; handles WebViewer init races). |
| `sendFullState(targetId)` | Unicast `sync_full` to a specific new joiner, carrying both `store` and `meta`. |
| `paramStr(name, fallback)` / `paramBool(name, fallback)` | URL-param helpers for the CONFIG block. |

---

## Payload / message protocol

### Standard envelope (used in all directions)

```json
{ "cmd": "<command>", "data": { ... }, "token": "<uuid>" }
```

- `cmd` — the operation name (string)
- `data` — operation parameters (object, may be `{}`)
- `token` — UUID used to correlate async request/response pairs

### FM → App (via `Perform JavaScript in Web Viewer`, function `fmCommand`)

| `cmd` | `data` fields | Description |
|-------|--------------|-------------|
| `add` | `key`, `value` | Create a new entry (or overwrite) |
| `patch` | `key`, `value` | Update an existing entry |
| `delete` | `key` | Remove an entry |

### App → FM (via `FileMaker.PerformScript(FM_SCRIPT, payload)`)

Default `FM_SCRIPT` is `fm-broadcaster.event_handler`; can be overridden via `?fmScript=`.

| `cmd` | `data` fields | When |
|-------|--------------|------|
| `loaded` | `instanceId`, `store` | Once FM is ready (and after a sync_full if peers existed) |
| `onChange` | `key`, `oldValue`, `newValue` | After every store mutation, regardless of origin |

### BroadcastChannel bus messages (internal, peer-to-peer)

Same envelope plus `origin` (sender INSTANCE_ID) and optional `target` (recipient INSTANCE_ID).

| `cmd` | Direction | Description |
|-------|-----------|-------------|
| `request_sync` | broadcast | New instance joining, asking for current state |
| `sync_full` | unicast (targeted) | Full store snapshot sent to a specific joiner |
| `add` | broadcast | Key/value was added |
| `patch` | broadcast | Key/value was updated |
| `delete` | broadcast | Key was removed |

---

## onfmready.js integration rules

- It is loaded as a **sibling file** via `<script src="onfmready.js">`. Both files must
  be deployed to the same URL path so the relative include resolves.
- It **must** be the first `<script>` in the document.
- Do not call `window.FileMaker` directly to detect FM context; use the `filemaker-expected`
  event (`event.filemaker` boolean) instead, as onfmready provides a fallback `FileMaker`
  object even outside FM.
- `filemaker-ready` fires only when FM actually injected. `filemaker-expected` fires in both
  contexts. The init sequence uses both:
  - `filemaker-expected` → start peer sync (`requestSync()`), set standalone status if outside FM
  - `filemaker-ready` → update status, send `loaded` if no peers responded

---

## Key constraints for agents

- **No build toolchain.** The file must remain self-contained. Do not add package.json,
  bundlers, transpilers, or external CDN links.
- **No server-side code.** Transport is `BroadcastChannel` only; durability is `localStorage`.
- **Vanilla JS only.** No frameworks. Keep it readable by FileMaker developers.
- **`applyMutation` is the only function allowed to write to `store` for live mutations.**
  The one exception is the empty-store fast path inside `mergeFullState`, which intentionally
  bypasses meta comparison to guarantee late joiners fill from peer snapshots.
- **`callFM` is the only function allowed to call `FileMaker.PerformScript`.** Do not scatter
  raw PerformScript calls through the codebase.
- **All persistence goes through `persist()` / `loadPersisted()`.** Do not call
  `localStorage.setItem` directly elsewhere; this keeps the `PERSIST` toggle effective and
  storage-key handling centralised.
- **All config knobs live in the CONFIG block.** New configurable values must be added there
  with a `paramStr` / `paramBool` URL-param override; do not introduce ad-hoc reads of
  `window.location.search` elsewhere.
- **Tokens must always flow through**: when FM sends a token in a command, that same token
  should appear in the `onChange` reply so FileMaker can correlate the async call.
- **`<script src="onfmready.js">` must remain the first script tag.**

---

## Verification (manual — no test suite)

1. Open `index.html` in two browser tabs (served from localhost, not `file://`).
2. Add a key/value in tab A → appears in tab B.
3. Edit a value in tab B → updates in tab A.
4. Delete a key in tab A → removed from tab B.
5. Open a third tab after entries exist → receives full store via `sync_full`.
6. **Persistence check**: close all tabs, then open one fresh → store still populated
   from `localStorage`.
7. **Persistence opt-out**: open `index.html?persist=0` → nothing read/written to
   `localStorage` for that WebViewer.
8. **Channel isolation**: open `index.html?channel=foo` and `index.html?channel=bar` →
   they do not see each other.
9. Open in FileMaker WebViewer → check the FileMaker script (default
   `fm-broadcaster.event_handler`, or whatever `?fmScript=` overrides to) receives the
   JSON payloads.

---

## Versioning

- Version is tracked in **two places** that must stay in sync:
  1. `VERSION` file at repo root (single line, e.g. `0.1.27`)
  2. `FM_BROADCASTER_VERSION` constant near the top of the main `<script>` block in `index.html`
- The version format is `MAJOR.MINOR.BUILD`.
- **After every commit, bump the BUILD number** (the third digit) and update both references in the same commit.
- **Do NOT increase MAJOR or MINOR versions unless explicitly instructed by the user.** Only the BUILD number changes automatically.
- The constant is logged to the console on load (`FM-broadcaster vX.Y.Z loaded`) — verify the message reflects the new value before committing.

---

## Commit style

- Imperative mood, present tense (`Add`, `Fix`, `Refactor`).
- One logical change per commit.
- Do not commit `old/`, `.DS_Store`, or any generated artifacts.
