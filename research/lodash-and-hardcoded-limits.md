# Lodash usage & hardcoded limit audit

Audit date: 2026-07-31 (branch `hoeken`). Follow-up to the `getCachedDeltas`
lodash removal (`perf(server): drop lodash from getCachedDeltas`). Two
questions: where else is lodash used (it is banned on hot paths per
AGENTS.md), and where else do hardcoded size/count limits thrash or silently
drop data when real-boat path/client/delta counts exceed them?

## Key insight: lodash's 500-entry path cache is process-global

The 500-entry memoize behind the original bug lives inside lodash's
`stringToPath` and is **shared by every `_.get`/`_.set` call in the whole
process**. On overflow it is **cleared wholesale**, not evicted. Fixing one
consumer doesn't fix the others — all remaining callers share (and thrash)
the same 500-entry budget.

## Tier 1 — same bug, still live (fix first)

1. **`src/interfaces/plugins.ts:419` — `app.getSelfPath()`** is
   `_.get(theApp.signalk.self, aPath)`. The most-called plugin API helper —
   derived-data, autopilot, and alarm plugins call it per delta with
   hundreds-to-thousands of distinct paths. Same bug as `getCachedDeltas`,
   arguably hotter. `getPath` at `plugins.ts:429` shares the key space and
   adds pressure to the same budget.
2. **`src/subscriptionmanager.ts:592-593` — `checkPosition`** uses
   `get(app.signalk.root, normalizedDelta.context)` and
   `get(vessel, 'navigation.position')`. This is the per-delta filter for
   every radius (`relativePosition`) subscription and keys the cache by AIS
   context strings (`vessels.urn:mrn:imo:mmsi:…`). A busy harbour exceeds
   500 distinct vessels → guaranteed wholesale clears on every delta of
   every radius subscriber. Fix: `vessel?.navigation?.position` plus a
   split-once context lookup.
3. **`src/put.ts:114,286,687`** — lodash `get`/`set` per PUT request.
   Request-rate, so it won't thrash alone, but line 286 keys by
   `context + '.' + path` (cardinality = contexts × paths) and pollutes the
   shared budget.
4. **`src/deltacache.ts:200-203`** — not lodash, but the same failure shape:
   `cachedContextPaths` (per-delta `context+path → parts` split cache in
   `getContextAndPathParts`) is **flushed entirely every 5 minutes**. At
   100+ deltas/sec with thousands of context×path pairs, the whole path
   space is re-split in a burst every 5 minutes. Replace with per-context
   eviction tied to the existing `deleteContext`/`pruneContexts`
   (`deltacache.ts:814`/`:823`).

## Tier 2 — hardcoded caps that drop or truncate real boat data

- **`packages/streams/src/liner.ts:30`** — partial (unterminated) lines over
  **2048 chars** are silently discarded (console.error only, no counter) for
  all TCP/UDP/serial/Execute providers. NMEA0183 is safe; JSON/CSV lines
  from canboat-csv, W2K-1, and Execute providers routinely exceed 2 KB.
- **`src/api/streams/binary-stream-manager.ts:21`** —
  `MAX_CONSECUTIVE_DROPS = 30` (~0.5 s at 60 Hz) hard-closes a radar
  WebSocket (`ws.close(1008)` at `:207`); half a second of marginal cabin
  WiFi is normal. `MAX_WEBSOCKET_BUFFER_SIZE = 256 KB` (`:16`) is half the
  delta path's 512 KB `BACKPRESSURE_ENTER`. Neither is configurable.
  Also `MAX_BUFFERED_FRAMES = 100` (`:26`) retains up to 100 multi-MB radar
  frames that are deliberately never replayed (comment at `:126-130`) —
  dead weight worth deleting.
- **WASM boundaries — silent truncation, no overflow check.** 64 KB
  response buffers in `src/wasm/bindings/resource-provider.ts:47`,
  `src/wasm/bindings/weather-provider.ts:97`,
  `src/wasm/bindings/radar-provider.ts:110`, and
  `src/wasm/loader/plugin-routes.ts:389`; `writtenLen` is trusted blindly,
  so a real waypoint database (> 64 KB) comes back as truncated JSON. 8 KB
  variants: `src/wasm/loader/plugin-routes.ts:205`,
  `src/wasm/loaders/standard-loader.ts:402`,
  `src/wasm/bindings/env-imports.ts:504`. Datagram/line truncation with no
  signal to the plugin: `src/wasm/bindings/env-imports.ts:824,980`. The fix
  pattern already exists in-tree: `src/wasm/wasm-serverapi.ts:357` checks
  overflow and logs.
- **`src/wasm/wasm-subscriptions.ts:172-176`** — 1000-delta buffer per WASM
  plugin during reload = 10 s of headroom at 100 deltas/sec; oldest deltas
  are FIFO-dropped (debug-only) so the plugin gets a gap. Same 1000-entry
  math for WASM sockets: `src/wasm/bindings/socket-manager.ts:82,550`.
- **`packages/streams/src/serialport.ts:39`** — outbound serial writes drop
  beyond 5 pending, behind a `debug()` only; no counter, not surfaced in UI.
- **`src/index.ts:171-176`** — `maxListeners = 50` is set on `app.signalk`
  only. `app` itself keeps Node's default of 10, but `src/interfaces/ws.ts`
  registers per-spark listeners on `app` (`serverlog` at `:1421`,
  `unitpreferencesChanged` at `:1349`) → spurious leak warnings at ~11
  admin-UI clients. Also `setMaxListeners` on `app`.
- **`src/interfaces/ws.ts:314`** — 30 WS connections per IP
  (env-overridable via `MAX_WS_CONNECTIONS_PER_IP`), but behind a reverse
  proxy with `trustProxy` unset every client resolves to one IP → hard 429
  at 31 connections server-wide.
- **`src/logging.js:11`** — 100 in-memory server-log lines (mirrored in the
  admin UI at `packages/server-admin-ui/src/store/slices/appSlice.ts:474`);
  a chatty plugin at boot scrolls provider failures out of view in under a
  second. Annoyance, not data loss.

## Checked and ruled benign

BackpressureManager thresholds (byte-based, env-overridable, coalesces to
latest-value-per-path rather than dropping, flags clients via
`$backpressure`); nmea-tcp thresholds (logged, escalate to disconnect);
HTTP rate limits (`serverroutes.ts:258`, only on access-request routes);
login throttle (`tokensecurity.ts:672`); pending-access-request cap of 100;
file-upload/icon/preset size caps; staleness sampling (`staleness.ts:31`,
intentional 10-sample median, config-overridable); npm search paging;
admin-UI pagination/clamps (tables are virtualized, not truncated).

## Suggested fix order

1. `src/interfaces/plugins.ts:419` (+`:429`) — drop lodash `get` from
   `getSelfPath`/`getPath` (split-once/cache, as done in
   `deltacache.ts:1459`).
2. `src/subscriptionmanager.ts:592-593` — lodash-free `checkPosition`.
3. `src/deltacache.ts:200` — replace 5-minute wholesale flush with
   per-context eviction.
4. The four 64 KB WASM `responseMaxLen` sites — loud overflow error instead
   of truncated JSON.
5. `packages/streams/src/liner.ts:30` — raise/configure the 2048 cap, count
   drops.
6. `binary-stream-manager.ts` — make drop/buffer thresholds configurable,
   align with delta-path backpressure; delete the never-read frame buffer.
7. `src/index.ts:171-176` — `setMaxListeners` on `app` too.

## Hot-path status (lodash-free already)

`src/streambundle.ts`, `src/interfaces/ws.ts`, `src/BackpressureManager.ts`,
`src/interfaces/rest.js` — no lodash. Cold lodash users (startup, config,
timers, admin routes) are harmless and only worth touching
opportunistically: mdns.js, pipedproviders.ts, modules.ts, index.ts,
config/*, appstore.js, applicationData.js, plugins.ts (non-API parts),
webapps.ts, deltaeditor.ts, deltastats.ts (interval only), security.ts:270,
serverroutes.ts, tokensecurity.ts (per-request), api/course,
notificationManager, gnssOffsetCorrector, admin-UI `lodash.remove`.
