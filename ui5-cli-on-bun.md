# UI5 CLI on Bun

This fork contains the Bun runtime-side changes needed to run the UI5 CLI end to end on Bun.

Sibling repo references:

- UI5 CLI fork: <https://github.com/ui5red/cli>
- Validation app: <https://github.com/ui5red/ui5-cli-on-bun>
- Local sibling checkouts used during development: `../cli` and `../ui5-cli-on-bun`

## What changed in this Bun fork

### 1. `process.binding("stream_wrap")` compatibility

Files:

- `src/bun.js/bindings/BunProcess.cpp`
- `test/js/node/process-binding.test.ts`

UI5 CLI's older HTTP/2 path and parts of the Node compatibility layer expect `process.binding("stream_wrap")` to exist.
This fork adds a minimal `stream_wrap` object with the fields needed by that code path and a regression test that verifies the shim is present and writable.

### 2. HTTP/HTTPS/TLS compatibility for Node-style server handling

Files:

- `src/js/node/_http_server.ts`
- `src/js/node/http.ts`
- `src/js/node/https.ts`
- `src/js/node/tls.ts`

These changes add Bun-side compatibility for Node server internals used by higher-level server stacks:

- a connection listener that wires sockets through the Node HTTP parser path
- export of `_connectionListener` through `node:http`
- a dedicated `https.Server` implementation that hooks secure connections into that listener
- TLS-side `instanceof` compatibility for Bun-created HTTPS server instances
- ALPN conversion handling for HTTPS server options

This work was necessary to get Bun's Node compatibility layer closer to the expectations of UI5 CLI's server stack and related middleware paths.

### 3. HTTP/2 server SETTINGS serialization fix

Files:

- `src/bun.js/api/bun/h2_frame_parser.zig`
- `test/js/node/http2/node-http2.test.js`

This was the main protocol-level blocker.

The issue was twofold:

- Bun server SETTINGS frames needed to omit `SETTINGS_ENABLE_PUSH`
- after reducing the logical payload size, Bun still wrote the full fixed buffer, leaking 6 trailing zero bytes onto the wire

That extra data desynchronized external HTTP/2 clients and caused failures such as GOAWAY errors blaming `DATA` on stream `0`.

The fix in this fork:

- writes server SETTINGS explicitly via `writeSettingsFramePayload(...)`
- computes the correct payload size via `settingsFramePayloadByteSize(...)`
- writes only the populated bytes via `stream.getWritten()` / `preface_stream.getWritten()`

The regression test `secure server should serialize initial SETTINGS frames without trailing bytes` validates the raw TLS exchange and makes sure the server:

- advertises a 36-byte SETTINGS payload
- does not emit setting id `2` (`SETTINGS_ENABLE_PUSH`)
- immediately follows with a valid SETTINGS ACK frame and no trailing garbage

## Result

With this fork, Bun can now successfully host the HTTP/2 server path needed by the UI5 CLI fork, including real external HTTP/2 clients rather than only Bun-to-Bun tests.

## Cross-repo picture

This fork provides the runtime support.
The sibling UI5 CLI fork contains the CLI-side integration work:

- JSDoc builder adjustments for Bun execution
- server-side Bun-specific `ui5 serve --h2` integration

See the sibling repo documentation for that side:

- <https://github.com/ui5red/cli/blob/main/ui5-cli-on-bun.md>

The standalone validation app provides the user-facing setup and test flow:

- <https://github.com/ui5red/ui5-cli-on-bun>

## Validation notes

Key validation performed against this fork:

- targeted Bun regression test for secure HTTP/2 SETTINGS serialization
- raw `h2c` validation with an external Node HTTP/2 client
- raw TLS `h2` validation with an external Node HTTP/2 client
- end-to-end validation through the sibling UI5 CLI fork serving a real fixture app over HTTP/2

## Recommended test flow

Use the standalone validation app as the entry point for setup and testing:

```sh
git clone https://github.com/ui5red/ui5-cli-on-bun.git
cd ui5-cli-on-bun
npm install
npm run setup:forks
npm run bun:build:fork
npm run smoke
```

That flow clones the sibling Bun and UI5 CLI forks automatically, prepares their dependencies, builds the custom Bun binary, and runs the end-to-end validation from one repository.
