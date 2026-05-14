# FM-broadcaster

A **zero-dependency, single-file micro-app** that gives every FileMaker Pro **Web Viewer**
panel in a shared layout a live, synchronised **key/value store** — with full CRUD access
from any panel or from FileMaker scripts, instantly, with no server.

---

## Table of contents

- [FM-broadcaster](#fm-broadcaster)
  - [Table of contents](#table-of-contents)
  - [What it does](#what-it-does)
  - [How it works](#how-it-works)
    - [Identity](#identity)
    - [Key/value store](#keyvalue-store)
    - [BroadcastChannel bus](#broadcastchannel-bus)
    - [Late-join sync protocol](#late-join-sync-protocol)
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
  - [Known limitations](#known-limitations)
  - [Roadmap / ideas](#roadmap--ideas)

---

## What it does

When `broadcast.html` is loaded in multiple FileMaker WebViewer panels simultaneously, all
instances share one real-time **key/value store**. Any panel — or a FileMaker script — can:

| Operation | How |
|-----------|-----|
| **Add** a key/value pair | UI input row, or FM `cmd: "add"` |
| **Patch** (update) a value | UI edit button, or FM `cmd: "patch"` |
| **Delete** a key | UI delete button, or FM `cmd: "delete"` |
| **Get notified** on any change | FileMaker script `fm-broadcaster.event_handler` receives `cmd: "onChange"` |
| **Know when a panel loaded** | Same script receives `cmd: "loaded"` with current store snapshot |

All changes propagate to every open WebViewer panel **and** to FileMaker within milliseconds.
Everything is purely client-side — no server, no database, no network round-trips.

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

All reads and writes go through `applyMutation(cmd, key, value, token, broadcast)`, which is
the single source of truth. It mutates `store`, re-renders the UI, optionally broadcasts on
the channel, and returns `{ oldValue, newValue }`.

### BroadcastChannel bus

```js
const channel = new BroadcastChannel('fm-broadcaster');
```

The `BroadcastChannel` API delivers messages to all browsing contexts that opened a channel
with the same name **and** share the same origin. In FileMaker, all WebViewers in the same
`.fmp12` file share an origin, making this work natively.

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

1. On load, the new instance broadcasts `request_sync`.
2. Every existing peer replies with a `sync_full` targeted at the new instance's ID.
3. The new instance accepts the first `sync_full` (they all carry the same state).
4. After receiving the snapshot (or after a 200 ms timeout if no peers exist), the instance
   calls `notifyFmLoaded()` to report its state to FileMaker.

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

Host `broadcast.html` on any static web server accessible to the FileMaker client machine
(local or remote). Set every Web Viewer object's URL to the same `http://` path so all
instances share the same origin.

```
http://localhost:8080/broadcast.html
```

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

// Query commands (read-only, no broadcast to peers)
{ "cmd": "get",    "data": { "key": "userName" },                    "token": "…" }
{ "cmd": "keys",   "data": {},                                       "token": "…" }
{ "cmd": "exists", "data": { "key": "userName" },                    "token": "…" }
{ "cmd": "count",  "data": {},                                       "token": "…" }

// Clear all keys (deletes everything, broadcasts to peers)
{ "cmd": "clear",  "data": {},                                       "token": "…" }
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

Else If [ $cmd = "onResult" ]
  # Response to a query command (get, keys, exists, count, clear)
  Set Variable [ $resultData ; Value: JSONGetElement($data ; "keys") ]
  If [ not IsEmpty($resultData) ]
    # keys command result
    Set Variable [ $keys ; Value: $resultData ]
  Else If [ not IsEmpty(JSONGetElement($data ; "count")) ]
    # count command result
    Set Variable [ $count ; Value: JSONGetElement($data ; "count") ]
  Else If [ not IsEmpty(JSONGetElement($data ; "cleared")) ]
    # clear command result
    Set Variable [ $cleared ; Value: JSONGetElement($data ; "cleared") ]
  Else If [ not IsEmpty(JSONGetElement($data ; "exists")) ]
    # exists command result
    Set Variable [ $exists ; Value: JSONGetElement($data ; "exists") ]
  Else
    # get command result
    Set Variable [ $key   ; Value: JSONGetElement($data ; "key") ]
    Set Variable [ $value ; Value: JSONGetElement($data ; "value") ]
    Set Variable [ $found ; Value: JSONGetElement($data ; "found") ]
  End If

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

**`onResult` payload examples:**
```json
// get
{ "cmd": "onResult", "data": { "key": "userName", "value": "Alice", "found": true }, "token": "…" }

// keys
{ "cmd": "onResult", "data": { "keys": ["mode", "userName"] }, "token": "…" }

// exists
{ "cmd": "onResult", "data": { "key": "userName", "exists": true }, "token": "…" }

// count
{ "cmd": "onResult", "data": { "count": 3 }, "token": "…" }

// clear
{ "cmd": "onResult", "data": { "cleared": 5 }, "token": "…" }
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
# then open http://localhost:8080/broadcast.html in two or more tabs
```

Outside FileMaker, `onfmready.js` detects that `FileMaker` was never injected and logs all
outbound script calls to the browser console instead of sending them to FM. This makes
browser-only development straightforward.

---

## onfmready.js

`broadcast.html` vendors **onfmready.js v2.1.11** (MIT — Stephan Casas) inline as the first
`<script>` block. It intercepts all calls to `FileMaker.PerformScript()` and queues them
until FileMaker injects its JS object, eliminating race conditions on macOS and WebDirect.

Key events it provides:

| Event | When | `event.filemaker` |
|-------|------|-------------------|
| `filemaker-expected` | Once FM injection window has passed | `true` if FM injected, `false` if not |
| `filemaker-ready` | Only when FM actually injected | — |

The app uses `filemaker-expected` to start the peer sync (`requestSync()`) regardless of
context, and `filemaker-ready` to confirm FM is live and set the status indicator.

Do **not** evaluate `window.FileMaker` directly to detect context — onfmready provides a
fallback object even outside FM. Use `event.filemaker` from `filemaker-expected` instead.

Source: https://github.com/stephancasas/onfmready.js  
CDN: `https://cdn.jsdelivr.net/npm/onfmready.js@2.1.11/dist/onfmready.min.js`

---

## Known limitations

| Limitation | Detail |
|-----------|--------|
| **Same-origin required** | All WebViewers must load from the same origin. `data:` URI instances get isolated origins and cannot communicate. Serve from a static host. |
| **No persistence** | The store lives in memory. Closing all WebViewers or quitting FileMaker wipes it. Persistence would require `localStorage`, IndexedDB, or writing to a FileMaker field. |
| **String values only** | Values are stored as strings. Structured objects should be JSON-serialised before storing. |
| **No access control** | Any WebViewer at the same origin/channel name can join and mutate the store. |
| **Concurrent patch race** | If two panels `patch` the same key simultaneously, the last broadcast write wins. No conflict resolution or CRDT logic is implemented. |
| **FM script name is hardcoded** | The script `fm-broadcaster.event_handler` must exist and be named exactly. A URL parameter to override this is a planned roadmap item. |

---

## Roadmap / ideas

- [x] Accept `?channel=` and `?script=` URL parameters to configure channel name and FM script name without editing the file
- [x] Optional `localStorage` persistence so state survives a panel reload (not a full FM quit)
- [x] Query API for FileMaker: `get`, `keys`, `exists`, `count`, `clear` commands with `onResult` responses
- [ ] Heartbeat / presence: detect and remove peers that closed without broadcasting a departure
- [ ] Structured value support: store arbitrary JSON objects, not just strings
- [ ] Conflict resolution hint: last-write-wins timestamp attached to each key
- [ ] `window.fmBroadcaster` public API for external scripts to call `add`, `patch`, `delete`, `getAll`
