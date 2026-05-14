# AGENTS.md — FM-broadcaster

Project-level conventions and context for AI agents (Devin, Copilot, etc.) working in this repository.

---

## What this project is

A **zero-dependency, single-file micro-app** (`broadcast.html`) designed to be loaded inside
FileMaker Pro **Web Viewer** panels — one instance per layout/window. Its purpose is to give
every simultaneously-open WebViewer a shared, real-time **key/value store** that any instance
can create, read, update, or delete. Mutations are reflected instantly across all open instances
and are reported to FileMaker via `FileMaker.PerformScript`.

No server. No build step. No npm.

---

## Repository layout

```
broadcast.html   — The active, production-ready implementation (self-contained HTML/CSS/JS)
.gitignore       — Excludes old/ and macOS artifacts
AGENTS.md        — This file
README.md        — Full developer + integrator documentation
old/             — Local-only archived prototypes (gitignored; do not touch)
```

`old/` is gitignored and should never be modified, referenced, or used as a source of truth.

---

## Source file: broadcast.html — logical sections

| Block | Purpose |
|-------|---------|
| `<script>` #1 — onfmready vendor | Inlined minified onfmready.js v2.1.11 (MIT). **Must remain the first `<script>` in the document.** Queues `FileMaker.PerformScript()` calls until the FM object is injected. |
| `<style>` | Self-contained UI styles. No external CSS. Uses plain values (no CSS variables) so it works standalone. |
| `<body>` markup | Static shell: header badge, peers row, add-row inputs, store table div, log div, edit modal. |
| `<script>` #2 — runtime | All application logic (identity, store, channel, FM bridge, UI). |

---

## JavaScript architecture

### Constants / identity

```
INSTANCE_ID     — crypto.randomUUID(), unique per WebViewer instance
INSTANCE_COLOR  — one of 7 preset hex colours, randomly picked
SHORT_ID        — last 6 chars of INSTANCE_ID, shown in badge
CHANNEL_NAME    — 'fm-broadcaster'  (BroadcastChannel namespace)
FM_SCRIPT       — 'fm-broadcaster.event_handler'  (FileMaker script name)
```

### State

```
store    — { [key: string]: string }   in-memory key/value store
peers    — { [instanceId: string]: color }   known live peer instances
```

### Key functions

| Function | Description |
|----------|-------------|
| `applyMutation(cmd, key, value, token, broadcast)` | **Single source of truth** for all store mutations. Mutates `store`, calls `render()`, optionally broadcasts on the channel. Returns `{ oldValue, newValue }`. |
| `callFM(cmd, data, token)` | Central choke-point for all outbound `FileMaker.PerformScript` calls. Builds the standard `{cmd, data, token}` payload. |
| `notifyFmLoaded()` | Sends `cmd: "loaded"` to FM with current `store` snapshot. |
| `notifyFmChange(key, old, new, token)` | Sends `cmd: "onChange"` to FM with change details. |
| `window.fmCommand(payloadStr)` | **Entry point called by FileMaker** via `Perform JavaScript in Web Viewer`. Parses payload, dispatches to `applyMutation`. |
| `requestSync()` | Broadcasts `request_sync` on load (late-join protocol). |
| `sendFullState(targetId)` | Unicast `sync_full` to a specific new joiner. |

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

### App → FM (via `FileMaker.PerformScript("fm-broadcaster.event_handler", payload)`)

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

- It is **vendored inline** (not fetched at runtime) so the file works from `file://` and
  data URIs without network access.
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
- **No server-side code.** Transport is `BroadcastChannel` only.
- **Vanilla JS only.** No frameworks. Keep it readable by FileMaker developers.
- **`applyMutation` is the only function allowed to write to `store`.** Do not mutate `store`
  directly elsewhere. This ensures render and broadcast stay consistent.
- **`callFM` is the only function allowed to call `FileMaker.PerformScript`.** Do not scatter
  raw PerformScript calls through the codebase.
- **Tokens must always flow through**: when FM sends a token in a command, that same token
  should appear in the `onChange` reply so FileMaker can correlate the async call.
- **onfmready vendor block must not be split or moved.** Keep it as-is at the top.

---

## Verification (manual — no test suite)

1. Open `broadcast.html` in two browser tabs (served from localhost, not `file://`).
2. Add a key/value in tab A → appears in tab B.
3. Edit a value in tab B → updates in tab A; console shows `app→FM onChange`.
4. Delete a key in tab A → removed from tab B.
5. Open a third tab after entries exist → receives full store via `sync_full`; console shows `loaded`.
6. Check console for `app→FM cmd=loaded` and `app→FM cmd=onChange` entries in all tabs.
7. Open in FileMaker WebViewer → check the FileMaker script `fm-broadcaster.event_handler`
   receives the JSON payloads.

---

## Commit style

- Imperative mood, present tense (`Add`, `Fix`, `Refactor`).
- One logical change per commit.
- Do not commit `old/`, `.DS_Store`, or any generated artifacts.
