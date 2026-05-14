# FM-broadcaster

[![JavaScript](https://img.shields.io/badge/JavaScript-ES6+-f7df1e?logo=javascript&logoColor=black)](https://developer.mozilla.org/en-US/docs/Web/JavaScript)
[![FileMaker](https://img.shields.io/badge/FileMaker-Pro%2019+-005577?logo=claris&logoColor=white)](https://www.claris.com/filemaker/)
[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com/fsans/fm-broadcaster)
[![MIT License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![onfmready.js](https://img.shields.io/badge/onfmready.js-2.1.11-orange.svg)](https://github.com/stephancasas/onfmready)

A **zero-dependency micro-app** that gives every FileMaker Pro **Web Viewer**
panel in the same FileMaker client a live, synchronised **key/value store** — with full
CRUD access from any panel or from FileMaker scripts, instantly, with no server. State
optionally persists across panel closes via `localStorage`.

---

## Table of contents

- [FM-broadcaster](#fm-broadcaster)
  - [Table of contents](#table-of-contents)
  - [What it does](#what-it-does)
    - [Typical use case](#typical-use-case)
  - [How it works](#how-it-works)
    - [Identity](#identity)
    - [Key/value store](#keyvalue-store)
    - [BroadcastChannel bus](#broadcastchannel-bus)
    - [Late-join sync protocol](#late-join-sync-protocol)
  - [Configuration](#configuration)
    - [Examples](#examples)
  - [Persistence](#persistence)
  - [Payload / message protocol](#payload--message-protocol)
    - [FM → App commands (`fmCommand`)](#fm--app-commands-fmcommand)
    - [App → FM commands (`fm-broadcaster.event_handler`)](#app--fm-commands-fm-broadcasterevent_handler)
  - [Transport layer](#transport-layer)
  - [FileMaker integration guide](#filemaker-integration-guide)
    - [Loading the micro-app](#loading-the-micro-app)
    - [FileMaker → App](#filemaker--app)
    - [App → FileMaker](#app--filemaker)
    - [Token-based async correlation](#token-based-async-correlation)
  - [Running standalone in a browser](#running-standalone-in-a-browser)
  - [onfmready.js](#onfmreadyjs)
    - [Key events it provides](#key-events-it-provides)
  - [Known limitations](#known-limitations)
  - [Roadmap / ideas](#roadmap--ideas)
  - [Credits](#credits)

---

## What it does

When `index.html` is loaded in multiple FileMaker WebViewer panels simultaneously, all
instances share one real-time **key/value store**. Any panel — or a FileMaker script — can:

| Operation | How |
|-----------|-----|
| **Add** a key/value pair | UI input row, or FM `cmd: "add"` |
| **Patch** (update) a value | UI edit button, or FM `cmd: "patch"` |
| **Delete** a key | UI delete button, or FM `cmd: "delete"` |
| **Get notified** on any change | FileMaker script `fm-broadcaster.event_handler` receives `cmd: "onChange"` |
| **Know when a panel loaded** | Same script receives `cmd: "loaded"` with current store snapshot |

All changes propagate to every open WebViewer panel **and** to FileMaker within milliseconds.
Everything is purely client-side — no server, no database, no network round-trips. State
also survives all panels being closed (within the same FileMaker client) thanks to the
built-in `localStorage` durability layer.

### Typical use case

- One WebViewer per layout, all loading the same `index.html` URL.
- Switching between layouts always exposes the same shared parameters.
- Two FileMaker `.fmp12` files opened in the same FileMaker client — both pointing
  WebViewers at the same hosted URL — share the same store across files.

---

## How it works

### Identity

On load each instance generates a unique ID:

```js
const INSTANCE_ID = crypto.randomUUID();  // e.g. "a3f7…-c82d"
```

This ID is stamped onto every bus message so instances can ignore their own echoes and
route point-to-point replies.

### Key/value store

```js
let store = {};   // { [key: string]: string }
```

All reads and writes go through `applyMutation(cmd, key, value, token, broadcast, meta)`,
which is the single source of truth. It mutates `store`, updates `storeMeta` (per-key
last-writer-wins metadata), re-renders the UI, persists to `localStorage`, optionally
broadcasts on the channel, and returns `{ oldValue, newValue, changed }`.

### BroadcastChannel bus

```js
const channel = new BroadcastChannel(CHANNEL_NAME); // default 'fm-broadcaster'
```

The `BroadcastChannel` API delivers messages to all browsing contexts that opened a channel
with the same name **and** share the same origin. In FileMaker, **all WebViewers in the
same FileMaker Pro client process** share an origin (for the same hosted URL), even across
different `.fmp12` files — so the store is shared across files in the same FM client.

### Late-join sync protocol

A newly opened panel has missed all previous mutations. The two-message handshake solves this:

```
New instance              Existing peers
     |                         |
     |── request_sync ────────>|  (broadcast)
     |<─ sync_full ────────────|  (unicast, targeted to new instance only)
     |                         |
  store populated           unchanged
     |
  notifyFmLoaded() ──────────────────> FileMaker
```

1. On load, the new instance broadcasts `request_sync` (retried periodically to handle
   races where a peer is still initializing).
2. Every existing peer replies with a `sync_full` targeted at the new instance's ID,
   carrying both `store` and `meta`.
3. If the new instance's local store is empty, it accepts the snapshot wholesale
   (fast path, no per-key meta comparison).
4. If the new instance already had local state (e.g. from `localStorage`), incoming
   keys are merged via `storeMeta` last-writer-wins (newer `ts` wins, ties broken by
   origin id).
5. After the snapshot (or after the retry window if no peers exist), the instance
   calls `notifyFmLoaded()` to report its state to FileMaker.

---

## Configuration

All runtime knobs live in a single `CONFIG` block near the top of the script, with
sensible defaults. Each can be overridden per-WebViewer via URL query parameters —
so the **same hosted file** can power multiple independent clusters in the same FM
client by using different channel names.

| Constant | URL param | Default | Purpose |
|----------|-----------|---------|---------|
| `CHANNEL_NAME` | `channel`     | `fm-broadcaster`              | `BroadcastChannel` namespace. Different value = isolated cluster. |
| `STORAGE_KEY`  | `storageKey`  | `fm-broadcaster:state`        | `localStorage` key holding `{store, meta}` JSON. |
| `FM_SCRIPT`    | `fmScript`    | `fm-broadcaster.event_handler`| FileMaker script that receives `loaded`/`onChange` payloads. |
| `PERSIST`      | `persist`     | `true`                        | When `false`, `localStorage` reads/writes are skipped. |

Boolean params accept `1 / 0 / true / false / yes / no` (case-insensitive).

### Examples

```
index.html
index.html?persist=0
index.html?channel=app-foo&storageKey=app-foo:state
index.html?fmScript=MyApp.broadcasterHandler
index.html?channel=app-foo&persist=0&fmScript=MyApp.handler
```

Use-cases:

- **Different `channel` per app** — keep two independent FileMaker apps from clobbering
  each other's KV store while running in the same FM client.
- **`persist=0`** — sandbox / experimentation panel that should not pollute storage.
- **`fmScript=...`** — route mutations to a different FileMaker handler script per file.

---

## Persistence

When `PERSIST` is true (default), every mutation also writes the full `{store, meta}`
blob to `localStorage` under `STORAGE_KEY`. On startup, the WebViewer restores from
`localStorage` **before** starting the peer-sync handshake. This means:

- A freshly opened WebViewer is immediately useful even when no peers are alive.
- Closing and reopening all WebViewers in the same FM client — or quitting and
  relaunching FileMaker on the same machine — preserves the store.
- Two `.fmp12` files in the same FM client, pointing WebViewers at the same URL, share
  the same persisted blob (same origin = same `localStorage`).

Failure modes (private mode, quota, disabled storage) are caught and silently ignored
— the app falls back to in-memory only.

---

## Payload / message protocol

All payloads — both the bus messages and the FileMaker bridge — use the **same envelope**:

```json
{
  "cmd":   "<command name>",
  "data":  { "key": "...", "value": "..." },
  "token": "550e8400-e29b-41d4-a716-446655440000"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `cmd` | string | The operation to perform |
| `data` | object | Parameters for the operation (may be `{}`) |
| `token` | UUID string | Correlation ID — pass it back in replies so FileMaker can match async calls |

### FM → App commands (`fmCommand`)

FileMaker calls JavaScript function `fmCommand` passing a JSON string:

| `cmd` | `data` fields | Description |
|-------|--------------|-------------|
| `add` | `key`, `value` | Add (or overwrite) a key |
| `patch` | `key`, `value` | Update an existing key |
| `delete` | `key` | Remove a key (`value` can be omitted / null) |

### App → FM commands (`fm-broadcaster.event_handler`)

The app calls `FileMaker.PerformScript("fm-broadcaster.event_handler", payload)`:

| `cmd` | `data` fields | When fired |
|-------|--------------|------------|
| `loaded` | `instanceId`, `store` | After FM is ready and initial store state is known |
| `onChange` | `key`, `oldValue`, `newValue` | After **every** store mutation (any origin — local UI, peer panel, or FM command) |

The `token` from the originating command is preserved in all reply payloads, enabling
FileMaker to correlate which script call produced which change notification.

---

## Transport layer

`broadcast.html` uses **`BroadcastChannel`** exclusively. Support matrix:

| Environment | Status |
|-------------|--------|
| FileMaker Pro 19+ (macOS / Windows) | Supported — Chromium engine |
| FileMaker WebDirect | Supported — modern browsers |
| Chrome / Edge / Firefox (standalone dev) | Supported |
| Safari 15.4+ | Supported |
| `file://` protocol | **Not supported** — `BroadcastChannel` requires same origin; serve from localhost |

---

## FileMaker integration guide

### Loading the micro-app

**Option A — hosted file (recommended)**

Host `index.html` (and the sibling `onfmready.js`) on any static web server accessible to
the FileMaker client machine (local or remote). Set every Web Viewer object's URL to the
same `http://` path so all instances share the same origin.

```
http://localhost:8080/index.html
http://localhost:8080/index.html?channel=my-app
```

The URL must match **exactly** across all WebViewers (same scheme, host, port, path) for
`BroadcastChannel` and `localStorage` to be shared.

**Option B — `data:` URI (single-panel only)**

Paste the file content into a FileMaker text field and load it as:

```
"data:text/html," & YourField
```

> **Warning:** `data:` URIs each get their own opaque origin. Instances loaded this way
> **cannot** communicate via `BroadcastChannel` — use Option A for multi-panel sync.

---

### FileMaker → App

Use the **Perform JavaScript in Web Viewer** script step (FileMaker 19+):

```
Perform JavaScript in Web Viewer [
  Object Name: "MyWebViewer" ;
  Function Name: "fmCommand" ;
  Parameters: JSONSetElement ( "" ;
    ["cmd"   ; "add"    ; JSONString] ;
    ["data"  ; JSONSetElement("" ; ["key";"myKey";JSONString] ; ["value";"hello";JSONString]) ; JSONObject] ;
    ["token" ; Get(UUID) ; JSONString]
  )
]
```

**Available commands:**

```json
// Add / overwrite a key
{ "cmd": "add",    "data": { "key": "userName", "value": "Alice" }, "token": "…" }

// Update an existing key
{ "cmd": "patch",  "data": { "key": "userName", "value": "Bob"   }, "token": "…" }

// Delete a key
{ "cmd": "delete", "data": { "key": "userName" },                    "token": "…" }
```

---

### App → FileMaker

Create a FileMaker script named exactly **`fm-broadcaster.event_handler`**.
It receives the payload as the script parameter (a JSON string):

```
# fm-broadcaster.event_handler
Set Variable [ $payload ; Value: Get(ScriptParameter) ]
Set Variable [ $cmd     ; Value: JSONGetElement($payload ; "cmd") ]
Set Variable [ $data    ; Value: JSONGetElement($payload ; "data") ]
Set Variable [ $token   ; Value: JSONGetElement($payload ; "token") ]

If [ $cmd = "loaded" ]
  # instance is ready; $data contains instanceId and full store snapshot
  Set Variable [ $store ; Value: JSONGetElement($data ; "store") ]
  # … process initial state …

Else If [ $cmd = "onChange" ]
  Set Variable [ $key      ; Value: JSONGetElement($data ; "key") ]
  Set Variable [ $oldValue ; Value: JSONGetElement($data ; "oldValue") ]
  Set Variable [ $newValue ; Value: JSONGetElement($data ; "newValue") ]
  # … react to the change …

End If
```

**`loaded` payload example:**
```json
{
  "cmd": "loaded",
  "data": {
    "instanceId": "a3f7c1d2-…",
    "store": { "userName": "Alice", "mode": "dark" }
  },
  "token": "550e8400-…"
}
```

**`onChange` payload example:**
```json
{
  "cmd": "onChange",
  "data": {
    "key": "userName",
    "oldValue": "Alice",
    "newValue": "Bob"
  },
  "token": "the-same-token-FM-sent-in-the-add-command"
}
```

---

### Token-based async correlation

Every payload carries a `token` (UUID). The lifecycle:

```
FileMaker script                    WebViewer
      |                                 |
      |── fmCommand({cmd:"add",         |
      |              token:"T1", …}) ──>|
      |                                 | applyMutation → broadcast → peers update
      |<── PerformScript("onChange",    |
      |     {token:"T1", key:…}) ───────|
      |                                 |
```

FileMaker can store `$token` before calling `fmCommand` and match it in the script
parameter when `onChange` fires — this confirms which set call produced which result,
even when multiple panels are mutating concurrently.

Tokens are generated with `crypto.randomUUID()` when FM does not supply one (e.g. for
mutations originating from the WebViewer UI).

---

## Running standalone in a browser

No build step is needed. Serve from localhost so `BroadcastChannel` gets a proper origin:

```bash
python3 -m http.server 8080
# then open http://localhost:8080/index.html in two or more tabs
```

Outside FileMaker, `onfmready.js` detects that `FileMaker` was never injected and logs all
outbound script calls to the browser console instead of sending them to FM. This makes
browser-only development straightforward.

---

## onfmready.js

**This project adopts [onfmready.js](https://github.com/stephancasas/onfmready) and strictly respects its original MIT-licensed copy.** The file is included verbatim as a sibling dependency and must be deployed alongside `index.html` without modification.

`index.html` loads **onfmready.js v2.1.11** (MIT — Stephan Casas) as the **first**
`<script>` tag, served as a sibling file from the same directory. It intercepts all calls
to `FileMaker.PerformScript()` and queues them until FileMaker injects its JS object,
eliminating race conditions on macOS and WebDirect.

**Deployment requirement:** Both files (`index.html` and `onfmready.js`) must be served
side-by-side at the same URL path. Do not modify `onfmready.js`; consume it as-is.

### Key events it provides

| Event | When | `event.filemaker` |
|-------|------|-------------------|
| `filemaker-expected` | Once FM injection window has passed | `true` if FM injected, `false` if not |
| `filemaker-ready` | Only when FM actually injected | — |

The app uses `filemaker-expected` to start the peer sync (`requestSync()`) regardless of
context, and `filemaker-ready` to confirm FM is live and set the status indicator.

Do **not** evaluate `window.FileMaker` directly to detect context — onfmready provides a
fallback object even outside FM. Use `event.filemaker` from `filemaker-expected` instead.

**Source:** https://github.com/stephancasas/onfmready  
**CDN:** `https://cdn.jsdelivr.net/npm/onfmready.js@2.1.11/dist/onfmready.min.js`

---

## Known limitations

| Limitation | Detail |
|-----------|--------|
| **Same-origin required** | All WebViewers must load from the exact same URL (origin + path). `data:` URI instances get isolated origins and cannot communicate. Serve from a static host. |
| **Same-machine only** | `BroadcastChannel` and `localStorage` are scoped to a single FileMaker Pro process. Sharing across different machines requires an external transport (FM Server, WebSocket relay, etc.). |
| **String values only** | Values are stored as strings. Structured objects should be JSON-serialised before storing. |
| **No access control** | Any WebViewer at the same origin/channel name can join and mutate the store. |
| **Concurrent patch resolution** | Conflicting writes are resolved by `storeMeta` last-writer-wins (timestamp + origin tiebreaker). Not a full CRDT — simultaneous writes still pick a winner deterministically rather than merging. |
| **localStorage quota** | Only meaningful for very large stores. Failure to persist (private mode, quota exceeded) is silently ignored; in-memory state still works. |

---

## Roadmap / ideas

- [x] Accept `?channel=`, `?fmScript=`, `?storageKey=`, `?persist=` URL parameters
- [x] Optional `localStorage` persistence so state survives panel close / FM relaunch
- [x] Last-writer-wins metadata for conflict resolution
- [ ] Heartbeat / presence: detect and remove peers that closed without broadcasting a departure
- [ ] Structured value support: store arbitrary JSON objects, not just strings
- [ ] `window.fmBroadcaster` public API for external scripts to call `add`, `patch`, `delete`, `getAll`
- [ ] Optional cross-host transport (FM Server / WebSocket relay) behind a feature flag

---

## Credits

**Created by** [Francesc Sans](mailto:air.fsans@gmail.com)

**FM-broadcaster** is released under the [MIT License](LICENSE).

Third-party dependencies:
- [onfmready.js](https://github.com/stephancasas/onfmready) v2.1.11 by Stephan Casas (MIT License)
