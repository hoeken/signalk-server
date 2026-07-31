# Static asset serving: caching & precompression

Research notes on reducing the cost of serving static assets (admin UI,
webapps, plugin UIs) — both HTTP round-trips and server-side CPU.

## Current state

All static file serving goes through `express.static` with **default
options** at six mounts:

| Mount | Source | Content |
|---|---|---|
| `/` | `src/interfaces/rest.js:114` | bundled `public/` assets |
| `/admin` | `src/serverroutes.ts:482` | admin UI (Vite build, content-hashed asset filenames) |
| `/documentation` | `src/serverroutes.ts:423` | built docs |
| `/<module>` per webapp | `src/interfaces/webapps.ts:90` (`mountWebModules`, also covers addons + embeddable webapps + plugin configurators) | installed webapps (Freeboard, KIP, ...) |
| plugin config UIs | `src/interfaces/plugins.ts:156` | plugin `public/` dirs |
| `/<pkg>` per WASM plugin | `src/wasm/loader/plugin-registry.ts:126` | WASM plugin webapps |

With default `serve-static` options, every response carries:

- `ETag` — weak, computed from file size + mtime (cheap, not a content hash)
- `Last-Modified`
- `Cache-Control: public, max-age=0`

Separately, the global `compression()` middleware (`src/index.ts:128`)
gzips every compressible 200 response **on the fly, per request**.

## Problems

1. **Revalidation storm.** `max-age=0` forces the browser to revalidate
   every asset on every page load. The server answers 304 (no body), but
   each conditional GET still traverses the full middleware stack —
   compression, helmet, cookie parsing, `http_authorize`
   (`src/tokensecurity.ts:760`) — plus an fs stat per asset.
2. **Repeated compression.** A client with a cold cache re-gzips the same
   webapp bundle on every visit. Node's zlib runs in the libuv threadpool
   (not the event loop), so this does not block the delta path, but it is
   real CPU/watts on ARM and contends with fs/crypto for the default 4
   threadpool threads.
3. **No precompressed-file support.** `express.static` never looks for
   `.br`/`.gz` sidecar files. A package that ships them today gets no
   benefit (and the sidecars are directly downloadable as opaque files).

What is already correct and must be preserved:

- Admin UI `index.html` is served with `no-cache, no-store`
  (`src/serverroutes.ts:2709`). It is templated per-request with addon
  `remoteEntry.js` script tags (`%ADDONSCRIPTS%`,
  `src/serverroutes.ts:441-472`) and must never be cached.
- Appstore npm metadata already gets `max-age=3600`
  (`src/interfaces/appstore.js:359`).

## Recommendation 1: per-mount Cache-Control

- **Hashed assets** (admin UI `assets/`, webapps built with hashed
  filenames): `{ maxAge: '1y', immutable: true }`. Content-hashed names
  make staleness impossible; revalidation requests disappear after first
  load.
- **Unhashed content** (`public/`, docs, plain-filename webapps): modest
  `maxAge` (e.g. `1h`). Bounded staleness; upgrades restart the server
  anyway, which is a natural staleness boundary.
- Templated HTML: keep `no-cache`.

Cost: one options object per mount. No dependencies, no memory.

## Recommendation 2: precompressed sidecar files

Replace `express.static` with `express-static-gzip` at the mounts above.
Per request it checks `Accept-Encoding`, serves `<file>.br` / `<file>.gz`
if present (with correct `Content-Encoding` and ETag), and falls back to
the plain file otherwise.

- The global `compression()` middleware **stays**: still wanted for
  dynamic JSON and as fallback for assets without sidecars. It skips
  responses that already have `Content-Encoding` set, so the two compose
  without double compression.
- Brotli beats gzip by ~15–20% on JS bundles and costs zero server CPU
  when precompressed at build time.

Sources for the sidecar files, in order of coverage:

1. **Admin UI build** — add compression (e.g. `vite-plugin-compression`)
   to `packages/server-admin-ui`. Covers the most-loaded assets for every
   installation.
2. **Individual webapps/plugins** — each package generates sidecars in its
   own build and ships them in the npm tarball. Fatter tarball; each
   maintainer opts in. See `research/plugin-loading-speedups.md`.
3. *(Deferred)* **Install-time generation** — the appstore install path
   could compress assets after `npm install`, covering packages that never
   opt in. More moving parts (install-time CPU on a Pi, cleanup on
   upgrade); revisit after 1+2 prove out.

## Do plugin-shipped `.br`/`.gz` files work automatically?

**Today: no.** `express.static` ignores sidecars, and `compression()`
re-gzips the plain file. Enabling them is a **server-side** change at the
mount points (one line for all standard webapps via
`src/interfaces/webapps.ts:90`). Once landed, the ecosystem contract is:
presence of sidecar files is the opt-in — no plugin API change, no
manifest flag.

Nuance: a plugin serving files through its own `registerWithRouter`
routes (`src/interfaces/plugins.ts:1074`) bypasses the static mounts, but
can already set `Content-Encoding: br` itself today; `compression()`
leaves already-encoded responses alone.

## What was considered and rejected

- **In-memory asset cache (JS-level `Map<path, Buffer>`).** Linux's page
  cache already keeps hot files in kernel memory; a Node-heap copy mostly
  duplicates it while adding GC pressure on Pi-class hardware. The real
  per-request costs are compression (fixed by sidecars) and revalidation
  (fixed by headers). Revisit only if profiling shows residual cost.
- **Threading/worker offload for static serving.** fs and zlib already run
  in the libuv threadpool; what remains on the main thread is HTTP
  parsing/routing/middleware JS, which workers cannot take over
  (`worker_threads` cannot receive sockets; HTTP/1.1 keep-alive mixes
  static and API requests on one connection, so request-level splitting
  must happen on the main thread). The legitimate "other thread" is a
  reverse proxy (nginx/Caddy) in front — valid per-installation, hard as
  an upstream default.

## Rollout sketch (separate PRs per repo conventions)

1. `express-static-gzip` + cache headers at server mounts
   (behavior-preserving for packages without sidecars).
2. Precompression in the admin UI build (`packages/server-admin-ui`).
3. Document the sidecar + hashed-filename conventions in
   `docs/develop/webapps.md`.
4. Measure: first-load / warm-load timings and server CPU before/after
   against a dev instance.
