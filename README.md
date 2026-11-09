# raidr_extension

A Chrome MV3 extension that captures a running web app — its network traffic,
its JavaScript, its source maps, its rendered DOM — and exports it as an
**raidr bundle** for reconstruction.

## Install

Not yet on the Chrome Web Store. Load it unpacked.

**Prerequisites**: [Bun](https://bun.sh) and Chrome 116+. The `@sudobility/*`
dependencies resolve from GitHub Packages (see [`.npmrc`](.npmrc)), which is
private, so `bun install` needs a GitHub token with `read:packages` scope —
e.g. add `//npm.pkg.github.com/:_authToken=<token>` to your user `~/.npmrc`, or
set it in the environment — before installing.

```bash
bun install
bun run build     # tsc && vite build → dist/
```

Then open `chrome://extensions`, enable Developer mode, and **Load unpacked**
pointing at `dist/`. Open the side panel on the tab you want to capture.

## What it captures

- **Network, via the DevTools Protocol.** A CDP session attached to the tab
  records requests and responses with their bodies — including the ones
  `webRequest` cannot see. Only `http(s)` is captured; a tab carries other
  extensions' traffic too, and it was ending up in shared bundles.
- **Chunks and source maps.** Every served script, plus the source map when the
  site shipped one, so reconstruction can start from real source rather than
  minified output.
- **Rendered DOM snapshots**, so server-rendered sites reconstruct correctly
  rather than as an empty SPA shell.
- **A coverage meter.** The side panel shows which routes and endpoints have
  been exercised, so you know what the capture is still missing before you
  export — gaps are recorded explicitly in the bundle rather than left implicit.

## Signing in to a site for raidr.app

On raidr.app, the API playground can ask the extension for a site's token so
you never copy it out of DevTools. The extension opens the site's sign-in page
in a popup window in your own browser profile, so an existing session is
picked up at once. It reads only that window's requests to the API host
raidr.app asked about, and accepts a token once a request carrying it to a
signed-in-only endpoint succeeds. It then closes the window and hands the
token to the raidr.app tab that asked, and nowhere else. If you close the
window first, nothing is returned, so a guest token is never taken.

Only raidr.app (and `localhost` for development) can make this request. This
is what the `webRequest` permission is for.

## Redaction happens before anything is written

Bearer tokens, cookies, auth headers, and values that look like credentials or
PII are redacted on the way into the buffer, not on the way out. Identifiers are
replaced with stable pseudonyms so a user id stays joinable across requests
without being the real one. The side panel reports exactly what was redacted.

The rules live in [`raidr_processor`](https://github.com/johnqh/raidr_processor), which the
extension imports — the same code path the CLI uses, so a bundle is redacted
identically no matter who produced it.

## Structure

```
src/
  background/       service worker: CDP session, request assembly, source maps
  offscreen/        capture buffer, IndexedDB store, export
  sidepanel/        React UI
  adapters/         thin interface over chrome.* for testability
  introspect/       page probes, serialized into the page via Runtime.evaluate
  bridge/           content script that lets raidr.app request a site token
```

## Development

```bash
bun run dev        # vite, HMR on 7178
bun run typecheck
bun test           # no browser needed
```

## The raidr project

| Repository | Role |
|---|---|
| [`raidr_processor`](https://github.com/johnqh/raidr_processor) | Bundle format and pure analysis |
| [`raidr_extension`](https://github.com/johnqh/raidr_extension) | Chrome MV3 extension that performs the capture — this repo |
| [`raidr_cli`](https://github.com/johnqh/raidr_cli) | Reconstruction CLI and the agent skill |
| [`raidr_web`](https://github.com/johnqh/raidr_web) | Landing site |

## License

BUSL-1.1 — see [LICENSE.md](LICENSE.md).
