# 06 — Wallet Integration (`window.ultra` + `@ultraos/wallet-sdk`)

**Last Updated:** 2026-07-24
**Read this to:** connect a dapp to the Ultra Wallet (browser extension or web wallet), sign
and broadcast transactions, handle events, and survive the known traps. The worked example
(`09`) applies everything here. Wallet *internals* (vault, signing lib, EBA keys) →
`[internal: ultraOS-doc web-browser-extension/WALLET_SYSTEM_AGENT_CONTEXT.md]`.
Note: `web-app/...` file:line citations below are into the private extension monorepo —
the npm `@ultraos/wallet-sdk` package (public) + this doc carry everything a dapp needs.

---

## 1. The ecosystem in one screen

- **Ultra Wallet browser extension** (Chrome MV3; source `web-app/apps/browser-extension-wallet`)
  injects a `window.ultra` provider into pages. It holds keys, serializes, signs, and
  broadcasts — the dapp never touches ABIs or private keys.
- **Web wallet** (`https://web-wallet.ultra.io`) — popup-based fallback when no extension is
  installed. No events support.
- **`@ultraos/wallet-sdk`** (npm) — the dapp-facing SDK that wraps both providers behind one
  API. **This is what your dapp should use** — not raw `window.ultra`.
- Published SDK version: **0.3.2** — **pin `^0.3.2` in new dapps** (0.3.1 and earlier can't be
  imported by plain Node / vitest / SSR; see `05` §1). Shipped dapps still pin `^0.3.1`/`^0.3.0`. The web-app source tree is ahead (0.5.0) — methods like `getNetworks`,
  `getAvailableAuthorizations`, attestation exist at HEAD but **verify against the installed
  version before using them**.

## 2. `window.ultra` — the provider surface

Injected as a frozen, tamper-proof Proxy by the extension's page-world script
(`web-app/apps/browser-extension-wallet/src/extension/inject.ts:61-81`), via manifest
`content_scripts` with `"world": "MAIN"` (`src/manifest.json:21-32`).

**Injection scope (critical local-dev fact):** the committed *source* manifest
(`src/manifest.json:11-31`) matches `https://*/*` **plus** `http://localhost/*` and
`http://127.0.0.1/*` (any port) — but **every production / QA build strips the loopback hosts**
via `apps/browser-extension-wallet/scripts/strip-loopback-hosts.mjs` (run by the
`build:browser-extension-wallet-{prod,qa}` npm scripts, mandatory for CWS). So the manifest that
actually ships to the Chrome Web Store has `content_scripts.matches = ["https://*/*"]` **only**.
**Net effect: the installed extension a tester downloads injects `window.ultra` on HTTPS only —
including on `localhost`.** Plain `http://localhost` gets no provider. The loopback matches
survive only in a **self-built, load-unpacked** extension: the strip is a separate step the
`build:browser-extension-wallet-{prod,qa}` npm scripts append *after* the nx build, so the §7.1
build command (`npx nx build …`, even `-c=production`) keeps loopback, while every CWS/QA artifact
built through those npm scripts is HTTPS-only. A dapp served over `http://` on any non-loopback
host never gets a provider on any build.
**Practical rule: to QA a local dapp against the real installed extension, serve it over HTTPS
(§7.2) — this is the norm, not an edge case.** Always feature-detect: `'ultra' in window`.

Methods: `connect, disconnect, signMessage, signTransaction, getChainId, purchaseItem,
getAccounts, getSelectedAccount, getAvailableAuthorizations, getNetwork, getNetworks,
switchNetwork, addNetwork` + EventEmitter (`on/off/once/...`). (`addNetwork` is a zombie —
its background route was removed; it returns `METHOD_NOT_FOUND` cleanly.)

## 3. `@ultraos/wallet-sdk` — how to use it

```bash
npm i @ultraos/wallet-sdk        # plus @wharfkit/antelope for chain reads (see 05)
```

```ts
import { UltraWalletSDK } from '@ultraos/wallet-sdk';

// Force the extension provider (the shipped dapps do this):
const sdk = new UltraWalletSDK({ provider: 'extension' });
// OR auto-detect (extension if `'ultra' in window`, else web-wallet popup),
// with a connect-time chain check that THROWS on mismatch:
const sdk2 = new UltraWalletSDK({ environment: 'testnet' });
```

Every call resolves to a response envelope — **it does not throw for user-level failures**:

```ts
interface UltraResponse<T> { status: 'success'|'fail'|'error'; data: T; message?: string; code?: number }
// codes: 4001 = user rejected; 4300 = web-wallet handshake timeout;
//        4301 = wallet popup blocked; -32604 unknown
```

Handle `4001` (user clicked Decline) everywhere you sign.
(Types: `web-app/libs/wallet-sdk/src/lib/interfaces/`.)

### 3.1 The canonical thin wrapper (copy this pattern)

`ultra-dex-dapp/src/ultraWallet.ts` — singleton SDK + availability guard + passthrough:

```ts
let sdk: UltraWalletSDK | null = null;
export function isAvailable() { return typeof window !== 'undefined' && !!(window as any).ultra; }
function getSDK() {
  if (!isAvailable()) throw new Error('Ultra Wallet extension is not installed');
  if (!sdk) sdk = new UltraWalletSDK({ provider: 'extension' });
  return sdk;
}
export const connect = (p = {}) => getSDK().connect(p);
export const signTransaction = (a: BlockchainTransaction[]) => getSDK().signTransaction(a);
export const dispose = () => { if (sdk) { sdk.dispose(); sdk = null; } };  // frees heartbeat+listener
// … disconnect/getNetwork/switchNetwork/getSelectedAccount/on/off passthroughs
```

### 3.2 Connect + silent session restore

```ts
// explicit connect (popup):
const res = await wallet.connect({});
// on app load — silent restore, no popup if origin already trusted:
try { const r = await wallet.connect({ onlyIfTrusted: true });
      if (r.status === 'success') await refreshFromWallet(); } catch { /* not trusted — fine */ }
```

After connect, read the truth from the wallet — the dapp never picks the account:
`getSelectedAccount()` → `{ accountName, permissions:[{name, publicKeys[]}] }`, and
`getNetwork()` → `{ name, chainId, nodeUrl }`; rebuild your read-only RPC client on the
wallet's `nodeUrl` (`ultra-dex-dapp/src/connection.ts:78-109`).

`ConnectResult`: prefer `selectedAccount` + `network.chainId`; `blockchainid`/`publicKey`
are deprecated legacy fields (fall back only for old wallets).

### 3.3 Sign + broadcast

**The wallet-sdk action shape is NOT the raw EOSIO shape.** SDK takes
`{contract, action, data, authorization}`; raw RPC / ultratest2 take
`{account, name, authorization, data}`. Mixing them up is a classic failure.

```ts
const actions = [{
  contract: 'eosio.token', action: 'transfer',
  authorization: [{ actor: account, permission }],       // StructuredAuthorization[]
  data: { from: account, to: 'ultra.dex',
          quantity: '1.00000000 UOS',                    // asset string, exact precision
          memo: `swap,${pairId},${minOut},${account}` },
}];
const res = await wallet.signTransaction(actions);       // signs AND broadcasts by default
if (res.status !== 'success') throw new Error(res.message || 'rejected');   // 4001 = declined
if (res.data.unsignedAuth?.length) throw new Error('not fully signed');     // partial-sign gap
const txId = res.data.transactionHash;
```

- Multi-action atomic tx = pass an array (e.g. the two add-liquidity transfer legs).
- `signTransaction(tx, { signOnly: true })` returns signatures without broadcasting.
- `signMessage(msg)` requires the message be prefixed `0x:`, `UOSx:`, or `message:`.
- The extension fetches ABIs + serializes itself — you pass plain JSON `data`.

### 3.4 Events

`WalletEventType = 'accountChanged' | 'networkChanged' | 'disconnect'` — there is **no
`chainChanged`**. Subscribe once on mount, tear down + `dispose()` on unmount
(`ultra-dex-dapp/src/connection.ts:158-180`):

- `accountChanged` — **don't trust the payload**; re-query `getSelectedAccount()`.
  `data.selected === null` means "no account on this chain" — do NOT wipe your state.
- `networkChanged` → re-read `getNetwork()`, adopt it, rebuild the read client. Guard with a
  `syncing` flag to prevent `switchNetwork ↔ networkChanged` ping-pong.
- `disconnect` → authoritative logout: clear local state, and **never call `disconnect()`
  back** (echo loop).
- Since SDK 0.3.0 the SDK manages background listener registration itself, with a 2 s
  heartbeat that survives extension service-worker restarts. Don't hand-roll
  `addExtensionListener`. Also: `on()` before `connect()` is silently dropped for untrusted
  origins — the SDK re-registers after connect for you.

## 4. Networks

| Network | chainId | RPC |
| --- | --- | --- |
| mainnet | `a9c481dfbc7d9506dc7e87e9a137c931b0a9303f64fd7a1d08b8230133920097` | `https://api.mainnet.ultra.io` |
| testnet | `7fc56be645bb76ab9d747b53089f132dcb7681db06f0852cfa03eaf6f7ac80e9` | see `07` (public testnet endpoints) |
| localhost | per-boot — resolve via `get_info` | `http://127.0.0.1:8888` |

- dapp→wallet switch: `switchNetwork(chainId)` (confirm popup; best-effort — the wallet may
  not have your localhost network). wallet→dapp: `networkChanged` event.
- SDK constructed with `{environment}` validates the wallet's chainId at `connect()` and
  throws `"Wallet environment mismatch…"`. Constructed with `{provider:'extension'}` and no
  environment, no check happens (the dex dapp's choice — it supports localhost).
- UX convention: on mismatch show a banner; don't force-switch.

## 5. Reads are NOT the wallet's job

Chain reads bypass the wallet entirely: keep a read-only `APIClient` from
`@wharfkit/antelope` pointed at the active network's RPC (details + snippets → `05` §3).
Re-point it whenever the wallet's network changes.

## 6. UX conventions (from the shipped dapps)

- Connect button: "Install Ultra Wallet" (disabled) when `!isAvailable()`; connect → show
  account name + Disconnect.
- Eager-connect on load with `onlyIfTrusted: true` (MetaMask-style).
- Refresh balances after connect and after every successful tx; keep last-known value on
  read failure.
- Tx flow: `busy=true` → run → success message with txId → reload on-chain state →
  `finally busy=false`. Surface `res.message` on failure; `4001` reads as "you declined".

## 7. Local development setup

### 7.1 Extension (build + load)

```bash
cd /home/adam/ultra.repos/web-app
npx nx build browser-extension-wallet --skip-nx-cache          # or -c=qa / -c=production
# → dist/browser-extension-wallet/  → chrome://extensions → Load unpacked
```

In the extension: add a custom network `http://127.0.0.1:8888` (loopback allowed without
HTTPS), import the local dev key, select the network. Well-known local dev keypair (chain
bootstrap default, see `04`):
priv `5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3` /
pub `EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV`. **Local/dev only — never on a
real network.**

### 7.2 HTTPS for manual QA

Vite 5's HTTPS dev server breaks on Node 22 (TLS HMR websocket). The shipped pattern:
build for prod, serve with the dependency-free `scripts/qa-https-server.mjs`
(`npm run qa:https` → `https://localhost:5181`, self-signed cert in `.certs/`). An HTTPS
page MAY read an `http://127.0.0.1:8888` RPC — loopback is exempt from mixed-content
blocking and the chain serves CORS `*`.

### 7.3 Local chain

Boot via ultratest2 `--keep-alive` (RPC on `:8888`) with an `e2e_setup.ts` that deploys your
contract and re-keys test accounts to the dev key — full recipe in `04` §6 and the worked
example. If the wallet needs account discovery (`get_accounts_by_authorizers`), the local
chain must run with `--enable-account-queries`.

## 8. Testing wallet flows

Two proven layers (details + file refs in `05` §6):

1. **Mocked `window.ultra`** (fast, headless, the default): Playwright `addInitScript`
   installs a mock implementing exactly the provider surface; `signTransaction` bridges to a
   Node-side signer that REALLY signs+pushes to the local chain with the dev key
   (`ultra-dex-dapp/tests/e2e/mockWallet.ts` + `chain.ts`). Assertions read chain tables
   directly in Node — the UI is checked against live on-chain truth.
2. **Real extension** (`ultra-tool-kit/tests/e2e/`): `chromium.launchPersistentContext` with
   `--load-extension` (MV3 ⇒ `headless:false`), poll `context.serviceWorkers()` for the SW,
   seed the vault via `sw.evaluate`, route-stub `**/v1/chain/**` so nothing escapes to the
   internet. Flaky on SW cold-start — keep retries.

`web-app/test-dapp-harness.mjs` is the extension's own dapp-integration regression harness
(connect/sign/disconnect/network/events) — useful as a reference for expected behavior.

## 9. Trap list (wallet-specific)

1. `window.ultra` missing → wrong scheme/host (HTTPS-only injection; §2) or extension not
   loaded. Feature-detect; never assume.
2. Two action shapes (SDK vs raw) — §3.3.
3. `UltraResponse` envelope — check `status`, don't try/catch for user rejection (`4001`).
4. `unsignedAuth` non-empty = partially signed tx — treat as failure.
5. Don't wipe state on `accountChanged` with `selected:null`; don't echo `disconnect`.
6. Web-wallet fallback has **no events** and a 10 s handshake timeout (`4300`).
7. `addNetwork` via SDK is dead — users add custom networks in the extension UI.
8. Asset strings must carry exact precision (`"1.00000000 UOS"`, 8 decimals for UOS).
9. Call `sdk.dispose()` on teardown — else the 2 s heartbeat timer leaks.
10. Localhost chainId differs per boot — resolve it from `get_info`, never hardcode.
