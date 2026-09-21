# Introducing hivexph-sdk v1.2.0 — A TypeScript SDK for Hive Custom JSON, Payments, and Token/NFT Creation

*Posted for PeakD — feel free to copy this markdown directly into the editor.*

---

Hello Hive! 👋

Today I'm excited to share something I've been building: **hivexph-sdk**, a reusable TypeScript SDK for the Hive blockchain — and its first public release on Hive, **v1.2.0**.

If you've ever wanted to watch blocks, send payments, parse `custom_json` operations, or create Hive Engine tokens and NFTs from your app without wiring up all the plumbing yourself, this SDK is for you.

```bash
npm install hivexph-sdk
```

- 📦 Package: `hivexph-sdk` (ESM only, fully typed)
- 🏠 Homepage: https://hivexph-sdk-frontend.vercel.app/
- 💻 Repository: https://github.com/rhiaji/hivex-sdk
- 📜 License: MIT

---

## What is hivexph-sdk?

A single, runtime-agnostic TypeScript SDK for Hive. It works in **Node 18+, Bun, Deno, browsers, and edge/Worker runtimes** — no React, no DOM APIs at module scope, and no side effects on import. Importing the package opens no connections and starts no streams.

Everything is reachable from one entry point. There are no deep import paths and no secondary entry points:

```
hive = new HiveClient(options)
 ├── configs         named configurations, account aliases, env references
 ├── accounts        key-free account references
 ├── rpc             blockchain communication
 ├── blocks.watch()  the single canonical block stream
 ├── reader          transaction reading + the unified stream engine
 ├── customJson      build / broadcast / watch standardized payloads
 ├── payments        native + Layer 2 payments with triggers
 ├── issuer          backend token + NFT operations
 ├── keychain        browser transactions through Hive Keychain
 └── keychainIssuer  browser token + NFT operations through Hive Keychain
```

---

## Quick start

```ts
import { HiveClient } from "hivexph-sdk";

const hive = new HiveClient({
  endpoint: "https://api.hive.blog", // optional; Beacon discovery otherwise
  accounts: {
    treasury: { accountEnv: "TREASURY_ACCOUNT", keyEnv: "TREASURY_ACTIVE_KEY" },
  },
});

// Read one transaction
const tx = await hive.reader.transaction({
  transactionId: "7b064a84a968caddd2496f3270f0cecafb954217",
});

// Watch standardized custom_json events
for await (const event of hive.customJson.watch({ id: "my-app" })) {
  console.log(event.account, event.action, event.metadata);
}
```

---

## One streaming engine

`hive.blocks.watch()`, `hive.customJson.watch()`, `hive.payments.watch()` and `hive.reader.stream()` all consume the **same** block reader — no namespace opens its own polling loop. The engine owns head tracking, sequential ordering, historical backfill, the gapless history-to-live transition, retries, and `AbortSignal` cancellation.

```ts
for await (const block of hive.blocks.watch({ fromBlock: 90_000_000 })) {
  for (const tx of block.transactions) {
    for (const op of tx.operations) {
      console.log(block.blockNumber, tx.transactionId, op.operationType);
    }
  }
}
```

Blocks are normalized once, so every consumer sees the same shape regardless of how a node represents operations.

---

## The standardized payload

Every transport (`custom_json`, native transfer memos, Hive Engine memos) carries the same shape:

```json
{ "action": "purchase_item", "metadata": { "itemId": "sword_01" } }
```

`action` is a required non-empty string, `metadata` is always present after normalization (possibly `null`), and there are no payload timestamps — **block time is the authoritative clock**.

---

## Payments

Send and validate native HIVE/HBD or Layer 2 payments with a consistent API:

```ts
await hive.payments.hive.transfer({
  from: hive.accounts.treasury,
  account: "bob",
  amount: "10.000",
  symbol: "HIVE",
  action: "purchase_item",
  metadata: { itemId: "sword_01" },
});

const result = await hive.payments.validate({
  transactionId, // the network is detected automatically
  expected: { account: "bob", symbol: "HIVE", quantity: "10.000", action: "purchase_item" },
});
// result.status: "pending" | "success" | "failed" | "invalid" | "not_found"
```

> ⚠️ Broadcasting a Layer 2 transfer only proves the `custom_json` reached Hive. Execution success is a separate sidechain check — always validate before crediting anything.

---

## Browser vs. backend

**Backend** issuers resolve an account alias, resolve the signing key lazily from the environment, sign, and broadcast:

```ts
await hive.issuer.token.issue({
  from: hive.accounts.treasury,
  symbol: "MYTOKEN",
  account: "alice",
  quantity: "10.000",
});
```

**Browsers** use the Hive Keychain API — no configuration, alias, or private key:

```ts
if (!hive.keychain.isAvailable()) throw new Error("Install Hive Keychain");

const signIn = await hive.keychain.requestSignIn({
  username: "alice",
  message: "Sign in to my-app: nonce-123456",
});
// Verify signIn.signature server-side to prove account ownership.
```

### Security rules

- Private keys never live in frontend code or committed configuration — use `keyEnv`.
- Keys are resolved lazily at signing time and never cached.
- Account references passed around the API are key-free.
- Config summaries and logs never contain secrets.

---

# 🆕 What's new in v1.2.0

This release is all about **creation flows**: sign-in with Keychain, and creating Hive Engine **tokens** and **NFT collections** — from both the backend and the browser — with safety preflights built in.

## Keychain sign-in

- `hive.keychain.requestSignIn()` for Hive Keychain account sign-in.
- Typed `KeychainSignInInput` and `KeychainSignInResult` exports.
- A Keychain playground sign-in tab for signing an app challenge message.

## Token creation (`tokens.create`)

- `hive.keychainIssuer.token.create()` and `hive.issuer.token.create()` for the Hive Engine `tokens.create` action.
- **Mandatory creation preflight:** the signing account must hold at least the sidechain token creation fee in BEE, and the token symbol must not already exist. Either failure throws (`INSUFFICIENT_BEE`, `TOKEN_ALREADY_EXISTS`) *before* Keychain opens or a key is resolved. Pass `skipChecks: true` to opt out.
- `checkCreate()` on both issuers plus the exported `TokenCreationChecker` for read-only fee, BEE balance, and symbol availability lookups.
- `buildCreate()` offline payload previews on `TokenActionBuilder` and both token issuers.
- New exports: `TokenCreateInput`, `TokenCreateActionInput`, `TokenCreationCheck`, `TokenCreationCheckInput`, `EngineTokenRow`, `KeychainTokenCreateInput`, the `BEE_SYMBOL` and token limit constants, and the `INSUFFICIENT_BEE` / `TOKEN_ALREADY_EXISTS` error codes.
- A token creation panel in the playground, covering both the Keychain path and the server path.
- Server-side token creation tests for payload parity, validation, preflight enforcement, and the `skipChecks` opt-out.

```ts
// Browser (Keychain)
await hive.keychainIssuer.token.create({
  username: "alice",
  symbol: "MYTOKEN",
  name: "My Token",
  precision: 3,
  maxSupply: "1000000",
});

// Backend (alias + env key)
await hive.issuer.token.create({
  from: hive.accounts.treasury,
  symbol: "MYTOKEN",
  name: "My Token",
  precision: 3,
  maxSupply: "1000000",
});
```

## NFT collection creation (`nft.create`)

- `hive.keychainIssuer.nft.create()` and `hive.issuer.nft.create()` for the Hive Engine `nft.create` action, following the same flow as token creation. `name` and `symbol` are required; `orgName`, `productName`, `maxSupply`, `website`, `authorizedIssuingAccounts`, and `authorizedIssuingContracts` are optional and omitted from the payload when absent.
- **NFT creation preflight:** the signing account must hold the NFT creation fee in BEE (100 BEE by default, read from `nft.params`) and the NFT symbol must not already exist. Either failure throws (`INSUFFICIENT_BEE`, `NFT_ALREADY_EXISTS`) before Keychain opens or a key is resolved. Pass `skipChecks: true` to opt out.
- `checkCreate()` and `buildCreate()` on both NFT issuers plus the exported `NftCreationChecker` for read-only fee, BEE balance, and NFT symbol availability lookups.
- New exports: `NftCreateInput`, `NftCreateActionInput`, `NftCreationCheck`, `NftCreationCheckInput`, `EngineNftRow`, `KeychainNftCreateInput`, the NFT creation fee and field limit constants, and the `NFT_ALREADY_EXISTS` error code.
- An NFT creation panel in the playground, covering payload preview, server and Keychain preflight checks, and Keychain broadcast.
- NFT creation tests for payload shape, field validation, preflight failures, and the `skipChecks` opt-out.

```ts
await hive.keychainIssuer.nft.create({
  username: "alice",
  name: "My Collection",
  symbol: "MYNFT",
  orgName: "My Org",           // optional
  productName: "My Product",   // optional
  maxSupply: "1000",           // optional, unlimited when omitted
  website: "https://example.com", // optional
});

// Read-only preflight, or an offline payload preview:
const check = await hive.keychainIssuer.nft.checkCreate({ username: "alice", symbol: "MYNFT" });
const payload = hive.keychainIssuer.nft.buildCreate({ name: "My Collection", symbol: "MYNFT" });
```

## Documentation

- The sign-in, token creation, and NFT creation flows are now documented in the README, API reference, and release notes.

---

# Full changelog

## 1.2.0

### Added

- `hive.keychain.requestSignIn()` for Hive Keychain account sign-in, with typed `KeychainSignInInput` / `KeychainSignInResult` exports and a playground sign-in tab.
- Token creation: `hive.keychainIssuer.token.create()` and `hive.issuer.token.create()` with a mandatory BEE fee + symbol preflight (`INSUFFICIENT_BEE`, `TOKEN_ALREADY_EXISTS`), `checkCreate()`, `buildCreate()`, `TokenCreationChecker`, related types/constants, a playground panel, and full tests.
- NFT creation: `hive.keychainIssuer.nft.create()` and `hive.issuer.nft.create()` with the same preflight pattern (`INSUFFICIENT_BEE`, `NFT_ALREADY_EXISTS`), `checkCreate()`, `buildCreate()`, `NftCreationChecker`, related types/constants, a playground panel, and full tests.

### Changed

- Documented the sign-in, token creation, and NFT creation flows in the README, API reference, and release notes.

## 1.0.0

First stable release. The public API is frozen under semver.

- **One canonical block engine** — `blocks.watch()`, `customJson.watch()`, `payments.watch()`, and `reader.stream()` are filtered views over a single polling loop.
- **One transaction reader** — payment parsing, Custom JSON parsing, and Layer 2 verification share one normalized operation model.
- **One write path per environment** — backend issuance through `hive.issuer`, browser issuance through `hive.keychainIssuer`, both emitting byte-identical contract actions from shared builders and validators.
- Pre-1.0 cleanup: removed the `Signer` system, executors, dry-run mode, legacy streams/builders, and all compatibility shims; `mint` naming is now `issue`.
- ESM build with generated type declarations; zero runtime Node built-ins and no DOM access at module scope.

---

## Try it out

```bash
npm install hivexph-sdk
```

The docs site with the full API reference and live playground is at **https://hivexph-sdk-frontend.vercel.app/**, and the source is on GitHub at **https://github.com/rhiaji/hivex-sdk**.

Feedback, issues, and ideas are very welcome — drop a comment below or open an issue on GitHub. Thanks for reading, and see you on-chain! 🐝

---

*#hive #development #typescript #sdk #hiveengine #nft #programming #opensource*
