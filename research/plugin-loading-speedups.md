# Speeding up plugin loading

Research notes on making plugins (and their web UIs) load faster.
Assumes the server-side static-serving fixes in
`research/static-asset-caching.md` (precompressed sidecars + cache
headers) have landed. Two distinct axes:

- **A. Browser-side**: how fast a plugin's webapp / config UI loads in the
  client.
- **B. Server-side**: how much installed plugins cost at server boot.

## A. Browser-side: plugin & webapp asset loading

### What plugin authors get for free after the server fixes

- Assets served from the standard mounts (`/<module>/...` via
  `mountWebModules`, plugin `public/` dirs) are served with cache headers
  and, where sidecars exist, precompressed bodies.
- The opt-in is file presence: ship `<asset>.br` / `<asset>.gz` next to
  the originals in the npm tarball. No API or manifest change.

### What plugin authors should do

1. **Generate sidecars at build time.** Webpack: `CompressionPlugin`
   (`brotliCompress` + gzip). Vite: `vite-plugin-compression`. Include the
   sidecars in the published package (`files` in package.json).
2. **Use content-hashed filenames** for JS/CSS bundles so the server can
   apply `immutable` caching. Unhashed assets get short-lived caching and
   still revalidate.
3. **Bundle diet** — the wins here dwarf transport tweaks for big webapps:
   - Code-split routes/panels so first paint doesn't parse the whole app.
   - Lazy-load heavy libs (charting, mapping) on first use.
   - Check for duplicated deps (each webapp bundles its own React etc. —
     unavoidable today, but worth auditing per package for accidental
     double-bundles).
4. **Plugins serving files via `registerWithRouter`**
   (`src/interfaces/plugins.ts:1074`) bypass the static mounts entirely.
   They can set `Content-Encoding: br` themselves today — the global
   `compression()` middleware skips already-encoded responses — but where
   possible prefer shipping static files in `public/` and letting the
   server mounts do the work.

### Admin UI addon scripts (module federation)

Every installed addon / embeddable webapp / plugin configurator gets a
`<script src="/<module>/remoteEntry.js">` tag injected into the admin UI's
`index.html` (`src/serverroutes.ts:441-472`). Consequences:

- Every admin page load fetches one `remoteEntry.js` **per installed
  package**, whether or not the user visits that plugin's config screen.
- `remoteEntry.js` files are served by the webapp mounts, so they benefit
  from sidecars + caching once landed — but they are typically *not*
  content-hashed (the filename is fixed by convention), so they only get
  short-`maxAge` caching, not `immutable`.
- Possible follow-up (not costed here): lazy-load federated modules when
  the relevant admin route is opened, instead of eagerly in `index.html`.
  Needs admin-UI changes and probably a fallback for older packages.

## B. Server-side: plugin cost at boot

### Current behavior (verified in code)

1. Discovery scans installed modules by npm keyword
   (`modulesWithKeyword`, `src/interfaces/plugins.ts:369-373`).
2. All plugins register concurrently in one `Promise.all`
   (`src/interfaces/plugins.ts:385`).
3. **Every installed plugin is imported AND instantiated at boot, even
   when disabled**: `importOrRequire(moduleDir)` and
   `pluginConstructor(appCopy)` run in `doRegisterPlugin`
   (`src/interfaces/plugins.ts:913-915`) before the
   `startupOptions.enabled` check (`src/interfaces/plugins.ts:1020`).
   The instance is needed to learn `plugin.id`, schema, and metadata for
   the config UI — but it means a disabled plugin still pays full
   `require()` + constructor cost (and pulls in its dependency tree) on
   every server start.

### Options, roughly by value/effort

1. **V8 compile cache (cheap, broad).** Node ≥22 supports an on-disk
   compile cache: `module.enableCompileCache()` early in startup, or
   `NODE_COMPILE_CACHE=<dir>` in the environment (server runs Node 24).
   Caches compiled bytecode for *all* CJS/ESM modules — server + every
   plugin — cutting parse/compile time on warm boots. No behavioral risk;
   ideal first experiment. Measure boot time before/after on Pi-class
   hardware.
2. **Defer import of disabled plugins (bigger, needs design).** To skip
   `importOrRequire` for disabled plugins, the server needs `plugin.id` and
   the config schema *without* constructing the plugin. Options:
   - Cache `{id, schema, metadata}` per package+version on first
     registration; on later boots, if the plugin is disabled and the
     cached version matches, register a stub from cache and import lazily
     on enable. (Enable/disable already restarts the plugin, so lazy
     import fits the existing lifecycle.)
   - Convention for `id` is usually derivable (most plugins use the
     package name), but it is set by plugin code — a cache is safer than a
     convention change.
   Payoff scales with how many installed-but-disabled plugins an
   installation carries; server logs the per-boot cost today only
   implicitly (no timing), so add measurement first.
3. **Per-plugin load timing (measurement, do first).** Wrap
   `registerPlugin` with timing and expose it via debug log
   (`signalk-server:interfaces:plugins`) so slow plugins are visible.
   Cheap, and turns options 1–2 from guesses into data.
4. **WASM plugins** load through a separate path
   (`registerWasmPlugin`, `src/interfaces/plugins.ts:530`) and are already
   gated behind the `wasm` interface flag; not the near-term bottleneck.

### Non-options

- **Threading the plugin `require()`s**: module loading must happen on the
  main thread to share the runtime; `Promise.all` already interleaves the
  I/O portions. The serial cost is V8 parse/compile — addressed by the
  compile cache — and each plugin's own constructor work.
