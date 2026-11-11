# Eightball WASM

Independent WASM clone of `lnbits/extensions/eightball` in this checkout. Extension ID: `eightballwasm`. Version 0.1.0, minimum LNbits 1.7.1.

## Features

Create, edit, delete and share Eightballs; newline-separated response lists; fixed sat prices; reusable bech32 LNURL QR; LNURL-pay metadata/callback and wallet message success action; optional in-page BOLT11 checkout/reveal; CSV export.

The browser uses the LNbits iframe bridge; server logic runs in a WebAssembly component. The extension requests no outgoing-payment permissions. All private records are owner-scoped. Public responses omit wallet credentials and internal configuration.

## Build and checks

Use an already installed `jco` compiler (validated with @bytecodealliance/jco 1.32.1), Node, and this checkout's Python environment. No new project dependencies are needed.

```sh
make check
make build JCO=/path/to/jco
make runtime-check
make package
```

`make check` runs domain regressions, JS syntax checks, and manifest checks. `make runtime-check` instantiates the compiled component through LNbits' real host/WIT adapters with isolated SQLite storage and FakeWallet invoices. It checks schema creation, two-owner isolation, missing permissions, invoice creation, settlement, and replay. The Python helper scripts are development tools; `make package` excludes them from the installable ZIP and runs LNbits' archive validator.

Sources are in `dev/src`; browser code is in `static`; schema and declarative migrations are in `storage`. Build directly in this directory. Each clone is self-contained.

## Installation

Install `dist/eightballwasm-0.1.0.zip` through the normal WASM installation/permission approval workflow. Use the accompanying SHA-256 file to check the release artifact. This package requires the approved host changes in this checkout: optional public-invoice LNURL metadata, the CSV/print parent bridge, and the read-only theme bridge. A stock 1.7.1 build without those changes does not provide all clone features.

After activation, management is at `/ext/eightballwasm`; public links are `/ext/eightballwasm/<id>`. The clone has separate storage and can coexist with its Python original. Existing Python records and URLs are not automatically migrated. Installation, activation and permissions in the running instance are separate from building this directory.

## API

Registered endpoints are under `/api/v1/ext/eightballwasm`; see `config.json` for exact routes, authentication and public owner-context sources. Normal operations return `{"ok":true,"data":...}` or `{"ok":false,"error":"..."}`; LNURL routes return protocol objects directly. Management lists are paginated in batches of 25 using `offset`. Authenticated requests use the LNbits account session, not Python extension wallet-key authentication.

Payment creation returns `attempt`, `payment_hash`, `payment_request`, `sats`, and expiry. `/payments/<attempt>` exposes payment status only (plus Eightball's response after settlement). Public request data never determines the receiving wallet or overrides the configured price. Public invoice URLs are bearer links and must be shared only with their intended recipients.

## Payment correctness and limits

Payment attempts snapshot the receiving wallet, amount and private data before invoice creation. Successful settlements validate extension, source, hash, amount and wallet binding. Receipts use the payment hash as their stable ID, so duplicate events cannot add the same payment twice. Deleting a source hides it but preserves outstanding invoice bindings and accounting records.

Public payment polling currently reads the recorded settlement events. The core event dispatcher has no durable retry queue, and the existing WASM API cannot independently look up a missed settlement. A restricted invoice-status host API is awaiting separate approval; until then this remains a recovery-parity gap with the Python originals.

The current storage host has no multi-operation transactions or compare-and-set. Invoice creation and receipt writes are not one transaction. A timeout after invoice creation can leave an unreturned invoice; it does not reverse payment. Receipts remain authoritative, and totals are computed from them rather than mutable counters. Aggregate scans are bounded at 20,000 records per source; CSV exports are bounded at 10,000 rows and 1 MiB.

The UI follows the original Quasar layouts, table columns and top dialogs, including the public payment pages. Render functions use the existing LNbits Vue/Quasar assets inside the sandbox without compiling Jinja or Vue templates. Colors, gradients and light/dark mode follow the active LNbits theme through the read-only theme bridge, including live changes. CSV and printing require a parent confirmation. Unlike the Python originals, public pages receive no wallet API keys. Prices and ownership are enforced by the server.

## LNURL

Creation stores the authenticated merchant's browser origin for callback URLs. Re-save the Eightball after moving the server to a different public origin. Discovery metadata is passed verbatim to the invoice host; LNbits hashes its UTF-8 bytes. Callback amounts must exactly match the configured price in millisatoshis. Each response is limited to the LNURL message limit of 144 characters. The protocol's message success action is included in the invoice callback, as in the Python original; it is not encrypted paid-content protection.
