# Delta hot path: CPU analysis

Research notes on where per-delta processing time goes, whether anything
synchronous should be async, and candidate optimizations. Based on a
read-through of the full path; **not yet profiled** — see "Next steps".

Assumed load model (from AGENTS.md): 100+ deltas/sec, 20+ WebSocket
clients, Pi 3–5 class hardware.

## Pipeline cost breakdown

For one delta with V values and S connected WS clients, in order:

1. **Decode (upstream packages).** canboatjs frame assembly + PGN decode,
   `N2kMapper.toDelta`. Bounded per message; not server code.
2. **`handleMessage`** (`src/index.ts:486`): `incDeltaStatistics`
   (cheap counters), `.map().filter()` over updates (two array
   allocations per delta), label-validation regex per update
   (`src/index.ts:526`), `getSourceId`, `new Date()` +
   `toISOString()` for missing timestamps.
3. **`dispatchDelta`** (`src/index.ts:338`) — the fanout core:
   - `deltaCache.ingestDelta` / per-value `onValue`
     (`src/deltacache.ts:241`): leaf walk + per-source write.
     Context-path `split('.')` results are already cached
     (`src/deltacache.ts:192-203`) — added historically because it
     showed in profiles.
   - `cloneDelta` on **every** delta (`src/index.ts:94-105`) for the
     `unfilteredDelta` emit (see finding 2).
   - `toPreferredDelta` (`src/deltaPriority.ts:585`): per-value map
     lookups; cheap when no priorities configured (see finding 4).
   - `app.signalk.addDelta` (FullSignalK, `@signalk/server-api`):
     per-value `path.split('.')` (uncached) + mutable full-model tree
     walk + sources-tree maintenance. Inherently per-value.
4. **StreamBundle fanout ×2** (`src/streambundle.ts:60` filtered,
   `:183` unfiltered): one `NormalizedDelta` allocation per value per
   family, each pushed onto 2–4 Bacon.js buses. Bacon wraps every value
   in event objects — high constant factor vs a plain callback list.
5. **Per-client delivery** (`src/interfaces/ws.ts:579-602`): per client
   per delta: `verifyWS` (throttled to 1/60s — fine), context check,
   `filterReadDelta`, backpressure check, then `spark.write(delta)` —
   Primus **JSON-serializes the delta independently for every client**.
   100 deltas/s × 20 clients = 2,000 near-identical `JSON.stringify`/s.

Majority of processing: split between stage 3+4 (per-value model/cache/
bus work — fixed cost per delta) and stage 5 (per-client serialization —
scales linearly with clients). Expectation: stage 5 dominates on any
installation with several open clients.

## Sync vs async: mostly the wrong lever

The path is CPU-bound synchronous JS by design; `async` would add
microtask overhead without removing work. There is no I/O to overlap:
sources-cache persistence is debounced onto timers, data logging goes
through async streams, WS socket writes are non-blocking with
`BackpressureManager` absorbing slow consumers.

Real exception — **synchronous stderr/stdout on the hot path**: Node
writes to a TTY/file synchronously. An enabled hot-path `debug` namespace,
or a chronically bad provider tripping per-delta `console.warn`
(e.g. the discard warning `src/index.ts:505`), blocks the event loop per
line. The repo's `debug.enabled &&` guard rule removes formatting cost,
but an *enabled* key still pays sync writes. Operational footgun, not a
bug.

## Findings, ranked

1. **N× JSON serialization per delta** (ws/Primus, stage 5). Largest
   expected CPU consumer; scales with clients. Fix shape: serialize once
   per equivalence class — most clients share
   `sourcePolicy=preferred` + no ACLs + same delta → stringify once,
   write the pre-encoded string to every matching socket. Per-user
   `filterReadDelta` only diverges when ACLs exist
   (`securityStrategy.anyACLs()` already exposes this). Medium effort,
   needs design (interaction with BackpressureManager accumulator and
   per-spark meta injection).
2. **Always-on unfiltered shadow pipeline.** `cloneDelta`
   (`src/index.ts:342`) + `pushUnfilteredDelta` per-value allocations run
   for every delta even with zero `sourcePolicy:'all'` consumers — and
   with no source priorities configured, `unfilteredDelta` is identical
   to `delta`. Roughly doubles per-value fanout cost in the common case.
   Fix: fast-path skip when no unfiltered consumers are registered.
3. **Bacon.js as the fanout mechanism** (stage 4 + subscriptions).
   Per-event wrapper allocations on every bus; `period` subscriptions run
   a lodash `reverse().uniqBy()` chain per flush per subscriber
   (`src/subscriptionmanager.ts:359-370`). Replacing Bacon is a big-bang
   refactor (plugins consume `app.streambundle` buses directly) — treat
   as known tax, not a near-term item.
4. **Priority-engine allocation on passthrough**
   (`src/deltaPriority.ts:612-660`): `update.values.reduce(...)` into a
   fresh array even when every value passes through unchanged — the exact
   pattern AGENTS.md forbids ("do not reduce into a new array when
   nothing was removed"), per update per self-context delta. Fix: detect
   no-change and return the original array.
5. **Wrapped-emitter spy per raw message**
   (`packages/streams/src/simple.ts:664-675` →
   `src/events.ts:260-275`): every incoming frame (pre-delta rate)
   emits `<providerId>-received` via `emitWithEmitterId`: bookkeeping,
   a template-string event name, and **two** `safeEmit` passes — even
   with no Admin UI data view attached (its only consumer). Fix:
   listener-count check before emitting.
6. Honorable mentions: `handleMessage` per-update regex + array churn;
   `FullSignalK.addDelta`'s uncached per-value `split('.')` (deltacache
   caches its equivalent); per-notification `setInterval` timers in the
   N2K transform (`packages/streams/src/n2k-signalk.ts:318`) can pile up
   on an alarm-happy bus.

Verified non-issues: `verifyWS` is throttled (60s,
`src/tokensecurity.ts:1438-1454`); delta input handler chain
(`src/deltachain.ts`) allocates one closure per delta but is otherwise
lean; `incDeltaStatistics` is trivial; dummy security's `filterReadDelta`
is a passthrough.

## Next steps

1. **Profile before fixing #1**: `node --cpu-prof` against a dev
   instance running sample N2K data with ~20 scripted WS clients;
   confirm the ranking above, especially serialization share.
2. Findings 2, 4, 5 are small, low-risk, independently PR-able without
   profiling (pure waste removal).
3. Finding 1 needs a short design note (equivalence-class keying,
   backpressure interaction) once profiling confirms its share.
