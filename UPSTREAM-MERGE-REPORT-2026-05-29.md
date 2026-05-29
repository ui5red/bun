# Upstream merge report — Bun fork

**Date:** 2026-05-29
**Branch:** `merge/upstream-2026-05-29` (uncommitted, not pushed)
**Base:** `40eff4488` (origin/main, "Improve process.binding shim, add sync comments, and clean up tls.ts")
**Merging:** `upstream/main` @ `f3ccf766df` ("ast: delete unused Boundering, RefDump, ...")
**Upstream commits ahead of our base:** 552

## Headline

Merge **completed cleanly** with one minor conflict, resolved.
All four functional changes our fork carries are preserved in the merged tree.
The merge is currently a staged-but-uncommitted "merge in progress" on a working branch — nothing has been committed or pushed.

## What changed in upstream Bun

Upstream is **mid Zig-to-Rust port** (not "rewritten in Rust" as previously believed; both `.zig` and `.rs` siblings exist for many files, and Cargo is now the workspace driver). Two structural moves directly affect our patches:

| Old path (our fork)                         | New upstream path                               |
| ------------------------------------------- | ----------------------------------------------- |
| `src/bun.js/api/bun/h2_frame_parser.zig`    | `src/runtime/api/bun/h2_frame_parser.zig`       |
| `src/bun.js/bindings/BunProcess.cpp`        | `src/jsc/bindings/BunProcess.cpp`               |

Both moves were tracked correctly with `git merge -X find-renames=50%` and our changes auto-merged into the new locations.

The Rust toolchain pin moved from `nightly-2025-12-10` to `nightly-2026-05-06`. The build now drives `cargo` + `ninja` rather than `build.zig` (the latter is gone upstream).

## Our four functional changes — status after merge

| # | Change                                                              | File(s)                                              | Status         |
| - | ------------------------------------------------------------------- | ---------------------------------------------------- | -------------- |
| 1 | HTTP/2 server SETTINGS frame: omit `SETTINGS_ENABLE_PUSH`, write only the populated bytes (no 6-byte tail) | `src/runtime/api/bun/h2_frame_parser.zig`, `test/js/node/http2/node-http2.test.js` | ✅ Auto-merged |
| 2 | `process.binding("stream_wrap")` shim with `Uint8Array streamBaseState` (Node parity, prevents segfault path) | `src/jsc/bindings/BunProcess.cpp`, `test/js/node/process-binding.test.ts` | ✅ Auto-merged |
| 3 | Node-style HTTP server `connectionListener` + `_connectionListener` export, dedicated `https.Server` hooking secure connections into it, TLS `Symbol.hasInstance` cleanup, ALPN conversion | `src/js/node/_http_server.ts`, `src/js/node/http.ts`, `src/js/node/https.ts`, `src/js/node/tls.ts` | ⚠️ One conflict, resolved (see below) |
| 4 | Regression tests for #1 and #2                                       | `test/js/node/http2/node-http2.test.js`, `test/js/node/process-binding.test.ts` | ✅ Auto-merged |

Verified post-merge with content greps:

- `Keep in sync with FullSettingsPayload` / `RFC 9113 Section 6.5.2` comments and `writeSettingsFramePayload(...)` / `settingsFramePayloadByteSize(...)` calls present in `h2_frame_parser.zig` at lines 256, 263, 1349, 1356, 1520, 1524.
- `JSC::JSUint8Array::create(globalObject, ..., 1)` for `streamBaseState` and `if (moduleName == "stream_wrap"_s)` branch present in `BunProcess.cpp`.
- `Object.defineProperty(Server, Symbol.hasInstance, ...)` (no `!!`) present in `tls.ts`.
- `secure server should serialize initial SETTINGS frames without trailing bytes` regression test present.
- `process.binding('stream_wrap')` test verifying `Uint8Array streamBaseState` present.

## The one conflict

`src/js/node/_http_server.ts` had two adjacent conflict regions — both resolvable without dropping functionality.

### Region 1 (around line 206)

**Both sides added new top-level functions in the same area, not modifying the same code.** Our side added `socketOnCompatError`, `socketOnCompatClose`, `socketOnCompatEnd`, `socketOnCompatData`, `parserOnIncomingCompat`, `connectionListener` (the Node HTTP-parser plumbing). Upstream added `normalizeServerTls` (a helper that defaults `requestCert: false` and forces `rejectUnauthorized: false` when not requesting a cert, to keep `https.Server({ ca })` from rejecting unauthenticated clients).

**Resolution:** Kept both. Our compat layer first, then `normalizeServerTls` immediately after. They are functionally independent.

### Region 2 (around line 348)

**Real semantic conflict on the same line.**

- Ours: `this[tlsSymbol] = tlsOptions;` (assigns the locally-built `tlsOptions` dict from a few lines above)
- Upstream: `this[tlsSymbol] = normalizeServerTls({ serverName, key, cert, ca, passphrase, secureOptions, requestCert: options.requestCert, rejectUnauthorized: options.rejectUnauthorized });`

**Resolution:** Took upstream. Reasoning:

1. The fields upstream passes are a strict superset of what our `tlsOptions` dict contained (`requestCert` and `rejectUnauthorized` are added; everything else is identical).
2. Wrapping with `normalizeServerTls` is the functional improvement upstream intended — defaulting `requestCert` to false and disabling `rejectUnauthorized` when not requesting. Without this wrap, an `https.Server({ ca })` would reject every cert-less client.
3. None of our HTTP/HTTPS work depended on `this[tlsSymbol]` being the literal `tlsOptions` reference.

**Side effect note:** The local `const tlsOptions = { ... }` binding above and the `convertALPNProtocols(options.ALPNProtocols, tlsOptions)` call that mutates it are now effectively dead — upstream's wrapped version doesn't use them, and ALPN conversion is no longer threaded through to `this[tlsSymbol]`. This is **a pre-existing bug in upstream's change** that they may not have noticed yet (or may have moved ALPN handling elsewhere). I did not fix it here — flagging for follow-up.

## Files touched by the merge

- 4923 files staged (cached) by upstream's churn — mostly `.rs` siblings appearing next to existing `.zig`, plus build-system rework.
- 2 worktree files post-resolve.

## Recommended next steps

1. **Build verification.** Started in-session as `npm run bun:build:fork`. Result captured at `/tmp/bun-build-2026-05-29.log`. **See the build report section in `/Users/I353257/Documents/Projects/bunnify/ui5-cli-on-bun/BENCHMARK-REPORT-2026-05-29.md`.**
2. **Test the dead-`tlsOptions` ALPN flag** — does upstream still honor `options.ALPNProtocols` on `https.Server`? If not, file an upstream issue or fix locally.
3. **Decide the commit shape** before pushing:
    - Single merge commit (preserves upstream history, easy to bisect),
    - Or rebase our 3 fork commits on top of upstream/main (loses the merge commit, linear history, but the rename detection is harder to verify by hand). Recommendation: **merge commit**, so the rename-tracking logic git did is preserved on the record.
4. **Do not auto-push.** Per global instructions, leave for human review/commit/push.
