# CLAUDE.md — raidr_extension

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.

MV3 Chrome extension that captures a running web app's network traffic and
runtime (chunks, source maps, DOM snapshots) and exports it as an **raidr
bundle** for reconstruction. Capture is driven from a side panel.

## Tech stack

- Vite + `@crxjs/vite-plugin`, **React 19**, TypeScript 6, **Bun** (never npm/yarn/pnpm)
- `@sudobility/components` + `@sudobility/design` for all panel UI; Tailwind
  driven by `createTailwindPreset()` from `@sudobility/design`
- `@sudobility/raidr_processor` — bundle format, redaction, coverage (redaction
  rules live there, not here)
- `@sudobility/raidr_types` — `extractCredential` and the raidr.app bridge protocol
- `fflate` for zipping; `fake-indexeddb` for tests
- `@sudobility/*` deps come from **GitHub Packages** (`.npmrc`); `bun install`
  needs a `read:packages` token. See README.

## Where it sits in the raidr family

- **`raidr_processor`** (npm `@sudobility/raidr_processor`; renamed from
  `raidr_lib` on 2026-09-30) — pure bundle-format / redaction / coverage lib.
- `raidr_cli` — reconstructs a project from a bundle; ships the agent skill.
- `raidr_crawler` — headless capture (non-extension path); holds a copy of `probes.ts`.
- `raidr_web` — landing page.
- Releases run from `raidr_app/scripts/push_all.sh` (after `raidr_processor`,
  dependency order). **There is no release/lint/format script in this repo.**

## Commands

All verified on 2026-11-08.

| Command | What it does | Status |
| --- | --- | --- |
| `bun install` | install deps (needs GitHub Packages token) | — |
| `bun run dev` | vite + crxjs, HMR on 7178 (strictPort), writes `dist/` | starts |
| `bun run build` | `tsc && vite build` → `dist/` (load unpacked) | passes |
| `bun run typecheck` | `tsc --noEmit` over `src` and `tests` | passes |
| `bun test` (or `bun run test:unit`) | 148 tests, 14 files, no browser | passes |

`bun test` is Bun's built-in runner; there is no `test` script in package.json.

## Structure

```
src/
  manifest.json     MV3 manifest; crxjs builds from it (vite.config.ts adds dev CSP)
  background/       service worker
    index.ts          top-level listeners, start/stop/resume, download
    cdpSession.ts     one attached tab: domains, bodies, navigations, probes
    requestAssembler.ts  joins CDP Network events per request id; http(s) only
    sourceMaps.ts     candidate map URLs, "is this map useful"
    tokenBridge.ts    wires TokenCapture to chrome.* (imported by index.ts)
    tokenCapture.ts   sign-in popup → verified site token for raidr.app
  bridge/           raidr.app ⇄ extension
    content.ts        content script on raidr.app/localhost: relays window messages
    origins.ts        BRIDGE_MATCHES, isBridgeOrigin, BridgeForward/BridgeReply
  offscreen/        capture buffer, IndexedDB store, export
    index.ts          message handler, throttled panel broadcast, export/zip
    sessionState.ts   per-session accumulator → coverage + BundleInput
    capturePipeline.ts  redaction boundary: redact, then hash+store bodies
    store.ts, hash.ts   content-addressed IndexedDB store (SHA-256 keys)
    viteManifest.ts   Vite lazy-chunk list from captured script bodies
    routeMatch.ts     `/users/:id` vs `/users/42`
    sameDomain.ts     registrable-domain filter applied at export
  sidepanel/        React UI (design-system components only)
    index.css         every design token the preset consumes (light + .dark)
  adapters/         thin interface over chrome.* for testability
  introspect/probes.ts  page probes, serialized into the page via Runtime.evaluate
  shared/messages.ts  RaidrMessage union + KINDS + isRaidrMessage guard
tests/              bun:test, mirrors src/; tests/support/FakeChromeAdapter.ts
```

## Capture pipeline

Side panel → worker → offscreen → export, over `chrome.runtime.sendMessage`
(broadcast; every listener filters by `kind`):

1. Panel sends `session/start` with the active tab id.
2. Worker (`background/index.ts`) refuses non-http(s) tabs, ensures the
   offscreen doc, sends `session/begin`, then `CdpSession.start` attaches
   `chrome.debugger` (Network/Page/Debugger/Runtime). Tab id + nav counter →
   `chrome.storage.session`.
3. CDP events hit the worker's **top-level** `adapter.onEvent` →
   `CdpSession.handleEvent`. `RequestAssembler` joins request/response/finish;
   bodies via `Network.getResponseBody` (else a `body-evicted` gap); source
   maps on `Debugger.scriptParsed`; DOM snapshot + probe re-run on navigation.
4. Worker's `CaptureSink` forwards each result as a `capture/*` message.
5. Offscreen `SessionState` ingests; `CapturePipeline` **redacts before
   storing** (bodies → IndexedDB by SHA-256). Stats/coverage/redaction
   broadcast to the panel at most every 250 ms.
6. Export: panel `export/start` (enabled only after redaction checkbox) →
   offscreen builds + zips `buildBundleFiles(state.bundleInput())` (filtered to
   the site's registrable domain) → `export/ready` → worker
   `chrome.downloads.download({ saveAs: true })`.

## Site token capture (raidr.app bridge)

raidr.app's API playground asks the extension for a site's token so the user
never copies it from DevTools. Protocol types come from `@sudobility/raidr_types`.

- Content script (`bridge/content.ts`, `document_start`) runs on `BRIDGE_MATCHES`
  origins, answers `ping` → `pong`+version, forwards `token/request|cancel` to
  the worker as `bridge/forward`.
- Worker (`tokenBridge.ts`) drops a forward unless the sender is this extension
  and the tab URL passes `isBridgeOrigin`, then calls `TokenCapture.start`.
- `TokenCapture` opens `loginUrl` in a 520×760 popup (user's own profile, so an
  existing session is picked up). `webRequest.onBeforeSendHeaders` reads only
  that window's requests to the requested `apiHost` via `extractCredential`;
  `onCompleted` accepts the token once a request to a `userPaths` path answers
  2xx (any 2xx if `userPaths` empty). Then closes the window, returns
  `token/result` to the requesting tab **only**.
- Window closed first → `token/failed closed` (guest token never taken).
- Watches live in `chrome.storage.session` (`raidr:token-watches`) so a
  recycled worker carries on.

## Hard-won constraints (do not regress)

- **Register chrome.* listeners at the top level of the service worker.** MV3
  restores only listeners registered during first evaluation; subscribing from
  inside a handler means capture dies silently when the worker recycles (~30s
  idle). Session state lives in `chrome.storage.session` so a restart resumes.
- **Only http(s) is captured.** A tab carries other extensions' requests too;
  they were ending up in shared bundles.
- **`src/introspect/probes.ts` must stay byte-identical** with the copies in
  `raidr_cli` and `raidr_crawler` (parity tests there gate releases via
  `push_all.sh`; skipped in single-repo CI). Any edit — comments/whitespace
  included — must be copied verbatim to both. It cannot move into
  `raidr_processor` (its tsconfig forbids DOM types). Probes are serialized with
  `.toString()` and run in the page, so they must reference nothing outside
  their own body (`tests/introspect` enforces this).
- **Redact before storing.** `CapturePipeline.ingest` is the only writer of
  bodies to the store, and redacts first. The per-session salt
  (`offscreen/index.ts`) is never persisted or exported.
- **`SessionState.begin` must reset every accumulating field.** The offscreen
  doc outlives a capture; an unreset field leaks one site's data into the next.
- **`BRIDGE_MATCHES` must equal `manifest.content_scripts[0].matches`**
  (`tests/bridge/origins.test.ts`). A new raidr_app deploy domain must be added
  to both and to `isBridgeOrigin`, or the app sees no extension.
- **The token bridge never answers a page outside `isBridgeOrigin`**, and reads
  only its own popup's requests to the one API host asked for. Keep both checks.
- **Every message kind must be in both the `RaidrMessage` union and `KINDS`**
  (`shared/messages.ts`), or `isRaidrMessage` drops it everywhere.
- **Design tokens: define every token the preset consumes** in both `:root` and
  `.dark` of `src/sidepanel/index.css` (`tests/messages.test.ts` checks the
  list). A mapped-but-omitted token resolves to *transparent* — the component
  lays out correctly and is simply invisible. Cards need an explicit border in
  light mode (`--card` and `--background` are both white). `ProgressBar` exists
  in `@sudobility/components` but is not re-exported; use `Progress`.

## Gotchas

- `src/manifest.json` version (`0.0.1`) is **not** synced with package.json; the
  built extension reports the manifest's value.
- Bridge messages (`bridge/forward`, `bridge/reply`) are **not** `RaidrMessage`
  kinds / not in `KINDS`; `tokenBridge.ts` and `content.ts` check `kind` themselves.
- The `webRequest` permission is used **only** by the token bridge. Capture goes
  through CDP (`chrome.debugger`).
- `capture/body` and `export/manifest` are declared kinds that nothing sends.
- DOM snapshot is taken only for same-document (client-side) navigations; full
  loads keep their served document.
- Export drops requests from other registrable domains on purpose; the panel's
  counts still include them.
- `bun run dev` writes a dev build into `dist/`; run `bun run build` before
  loading `dist/` for a real test.

## Making common changes

- **New message kind**: `RaidrMessage` union + `KINDS` (`shared/messages.ts`) →
  `tests/messages.test.ts` → sender → receiver(s).
- **New CDP capture**: `CdpSession.handle` (+`start` if a new domain) →
  `CaptureSink` method → `buildSink` → `capture/*` message → `SessionState`
  ingest (reset in `begin`) → `bundleInput` → tests.
- **New probe**: edit `introspect/probes.ts` + `PROBE_SOURCES`, copy the file
  byte-for-byte to `raidr_cli` and `raidr_crawler`, then use from `CdpSession`.
- **Redaction rule**: change `raidr_processor` and bump the dep here.
- **New raidr.app bridge origin**: `BRIDGE_MATCHES` + `isBridgeOrigin`
  (`bridge/origins.ts`) and `content_scripts[0].matches` (`manifest.json`).

## Testing

`bun test` runs everything under `tests/` with no browser: `chrome.debugger` is
replaced by `tests/support/FakeChromeAdapter.ts`, IndexedDB by
`fake-indexeddb/auto`, probe sources run via `new Function`.
`background/index.ts`, `offscreen/index.ts` and the React panel have no unit
tests (chrome.* wiring) — verify them by loading `dist/` unpacked and capturing
a real page.
