# 05 — Dapp Development

**Last Updated:** 2026-09-25
**Read this to:** build the web frontend for an Ultra contract the way the shipped dapps do.
Exemplars `[internal]`: `ultraio/ultra-dex-dapp` (the reference), `ultra-lend-dapp`,
`ultra-farm-dapp` (Vue); `ultra-bridge-dapp` (React, production at bridge.ultra.io).
Wallet specifics live in `06`; this doc is the app around it.

---

## 1. The proven stack

Vue 3 + Vite + TypeScript, `<script setup lang="ts">`. Deliberately minimal deps —
exactly three runtime packages:

```json
"dependencies": {
  "vue": "^3.5.0",
  "@ultraos/wallet-sdk": "^0.6.1",     // extension + Web Wallet signing; no keys in the dapp
  "@wharfkit/antelope": "^1.0.13"      // read-only chain client
}
```

**Pin `@ultraos/wallet-sdk` at `^0.6.1`.** 0.3.1 and earlier ship directory/
extensionless internal imports that Node's ESM resolver rejects
(`ERR_UNSUPPORTED_DIR_IMPORT`) — fine in a browser bundle, but it breaks **vitest, SSR and
any plain-Node script**, i.e. exactly the unit testing in §6.

**Dev toolchain that is known to work together** (an unpinned `npm i -D vite vitest` can
resolve to two different vite majors and hard-fail `vue-tsc`):

```json
"devDependencies": {
  "vite": "^5.4.0",
  "@vitejs/plugin-vue": "^5.1.0",
  "vitest": "^2.1.0",
  "vue-tsc": "^2.1.0",
  "typescript": "^5.6.0",
  "@playwright/test": "^1.55.0"   // E2E (§6); then `npx playwright install --with-deps chromium`
}
```

**`tsconfig.json` must list `"types": ["vite/client"]`** in `compilerOptions`. Without it
`import.meta.env` (`VITE_NODE_URL`, `import.meta.env.DEV`, …) has no type and `vue-tsc --noEmit`
fails with `Property 'env' does not exist on type 'ImportMeta'`. `vite build` alone never catches
this (it doesn't type-check); only the `npm run build` = `vue-tsc --noEmit + vite build` step (§7)
does — so it surfaces in CI/`build`, not `dev`. Add the line when you scaffold, not when CI breaks.
(Equivalently, a `/// <reference types="vite/client" />` line in a `src/vite-env.d.ts` — what
Vite's own `create-vite` scaffold generates; either satisfies the type, don't add both blindly.)

No state library — one exported `reactive({...})` state object. React + wagmi/RainbowKit
is the bridge-dapp variant if you need EVM too.

## 2. Project structure (replicate this)

```
src/
  ultraWallet.ts    # provider-aware SDK wrapper (extension singleton + Web instance per env)
  connection.ts     # reactive state + provider-gated lifecycle + read client + signAndPush
  config.ts         # contract names, NETWORKS[] (chainId+nodeUrl), tokens, matchNetwork()
  <domain>Client.ts # table reads + action builders for YOUR contract
  <domain>Math.ts   # math mirror of the contract (see §5)
  App.vue           # connect button, network selector, account badge
  components/*.vue  # feature panels
tests/e2e/          # extension mock + live chain; add Web popup/provider tests per 06 §9
scripts/qa-https-server.mjs   # HTTPS server for real-extension manual QA
src/__tests__/      # vitest (math mirror)
```

Key split: `ultraWallet.ts` selects and records the SDK provider; `connection.ts` owns app
state and branches between extension lifecycle and Web popup lifecycle; `<domain>Client.ts`
translates UI intents into actions/reads. Config defaults:
`VITE_NODE_URL` env overrides the read endpoint (`http://127.0.0.1:8888` for local chain).

## 3. Talking to YOUR contract

Two directions, two mechanisms:

- **Writes** = wallet-signed transactions. Build plain-JSON actions (wallet-sdk shape:
  `{contract, action, data, authorization}`) and hand them to `signAndPush` (`06` §5).
  For memo-dispatch contracts the "action" is an `eosio.token::transfer` **to** the
  contract with the routing memo — e.g. the DEX swap
  (`ultra-dex-dapp/src/dexClient.ts:60-139`):

  ```ts
  await signAndPush([{
    contract: 'eosio.token', action: 'transfer', authorization: auth(),
    data: { from: state.account, to: DEX_CONTRACT,
            quantity: toAssetString(amountIn, sym),        // "1.00000000 UOS"
            memo: `swap,${pair.id},${minOut},${state.account}` },
  }]);
  ```

- **Reads** = direct RPC with `@wharfkit/antelope`, no wallet involved:

  ```ts
  import { APIClient } from '@wharfkit/antelope';
  let client = new APIClient({ url: state.nodeUrl });
  const rows = await client.v1.chain.get_table_rows({
    code: 'ultra.dex', scope: 'ultra.dex', table: 'pairs.a', json: true, limit: 100 });
  const bal = await client.v1.chain.get_currency_balance('eosio.token', account); // ["980.00000000 UOS"]
  const info = await client.v1.chain.get_info();      // chain_id for localhost matching
  ```

  For extension, rebuild when its network changes. For Web Wallet, bind reads to the environment
  used to construct the Web SDK and rebuild only after an explicit environment change/reconnect.

Asset strings: always the exact on-chain precision (`"1.00000000 UOS"`). Parse balances by
splitting on the space.

## 4. State & sync model

One reactive state: `{ extensionAvailable, provider, connected, account, permission, chainId,
nodeUrl, balances, syncing, busy }`. Rules (from the shipped `connection.ts`):

- Retain `provider: 'extension'|'web'`; a missing extension must not disable Web Wallet.
- Extension: wallet is the source of account/network truth; subscribe after connect and silently
  reconnect only this provider with `onlyIfTrusted:true`.
- Web: account comes from `connect()`, network comes from construction environment + `getChainId()`;
  never call extension-only query/event/switch methods and never open it silently on mount.
- On extension `networkChanged`, adopt the wallet network and rebuild the read client. On Web
  environment change, disconnect, rebuild for the chosen environment, and require reconnect.
- `syncing` flag guards the switchNetwork↔event loop; `busy` serializes tx submission.

## 5. The math-mirror pattern (correctness-critical)

If your contract computes anything the UI must predict (prices, shares, rewards,
interest), port that math to TypeScript **bit-for-bit using BigInt** (`amm.ts`,
`lendMath.ts`, `farmMath.ts`) and vitest it against the contract's own test vectors.
The mirror is the only correctness-critical client code: UI previews, min-out/slippage
floors, and E2E assertions all come from it. Floor/round exactly like the contract; if the
contract uses u128 saturation, either mirror it or document the divergence.

## 6. Testing

> ⚠️ **Reading a contract assert out of a WharfKit error.** `err.message` is only the generic
> `eosio_assert_message assertion failure at /v1/chain/push_transaction`. Your contract's
> actual `check()` string is in **`err.response.json.error.details[].message`** — so
> `String(e.message)` makes every "expect this to revert" test vacuously pass. Extract with
> something like:
> `const m = e?.response?.json?.error?.details?.map(d => d.message).join(',') ?? e.message`.

- **Unit (vitest):** the math mirror. `npm test` = `vitest run`. ⚠️ **Scope vitest away from your
  Playwright specs** — vitest's default `include` matches `*.spec.ts` too, so `vitest run` will
  try to execute the E2E specs and crash importing `@playwright/test`. In `vite.config.ts` set
  `test: { include: ['src/**/*.test.ts'] }` (name unit tests `*.test.ts`, E2E `*.spec.ts`), or
  add `test: { exclude: ['tests/e2e/**', ...configDefaults.exclude] }`.
- **E2E (Playwright) against a REAL seeded local chain** — the shipped pattern:
  1. Terminal A: boot the chain — `ultratest2 … -t $PWD/e2e_setup.ts --keep-alive`
     (seeds contract + users re-keyed to the dev key; `04` §6).
  2. Terminal B: `npx playwright test` — config runs Vite (`webServer`, own port,
     `VITE_NODE_URL` set), `workers: 1`, sequential.
  - `tests/e2e/mockWallet.ts`: `page.addInitScript` installs a `window.ultra` mock
    implementing the exact provider surface; `signTransaction` bridges via
    `page.exposeFunction` to `tests/e2e/chain.ts`, which REALLY signs with the dev key and
    pushes (`@wharfkit/antelope`: fetch ABI → build Action/Transaction → sign →
    `push_transaction`), returning the SDK envelope (`{status:'fail', message}` on an assert) so
    contract asserts surface like the real wallet. Minimal `chain.ts` (clean-room validated,
    `@wharfkit/antelope@^1.0.13`; the key is the local-chain dev key of `04` §5, passed in via env):
    ```ts
    import { APIClient, Action, Transaction, SignedTransaction, PrivateKey } from '@wharfkit/antelope';
    const client = new APIClient({ url: process.env.VITE_NODE_URL ?? 'http://127.0.0.1:8888' });
    const key = PrivateKey.from(process.env.LOCAL_DEV_KEY!);   // local test chain only
    export async function signAndPush(txs: any | any[]) {   // { contract, action, data, authorization }
      try {
        const info = await client.v1.chain.get_info();
        const actions = await Promise.all([txs].flat().map(async (a) => Action.from({ account: a.contract,
          name: a.action, authorization: a.authorization, data: a.data },
          (await client.v1.chain.get_abi(a.contract)).abi!)));
        const tx = Transaction.from({ ...info.getTransactionHeader(120), actions });
        const sig = key.signDigest(tx.signingDigest(info.chain_id));
        const res = await client.v1.chain.push_transaction(SignedTransaction.from({ ...tx, signatures: [sig] }));
        return { status: 'success', data: { transactionHash: String(res.transaction_id) } };
      } catch (e: any) {
        return { status: 'fail', message: e?.response?.json?.error?.details?.map((d: any) => d.message).join(',') ?? String(e?.message ?? e) };
      }
    }
    ```
    `mockWallet.ts` exposes it with `page.exposeFunction('__sign', signAndPush)` and has the
    `addInitScript` mock's `signTransaction(tx)` return `window.__sign(tx)`; `getChainId` returns
    `get_info().chain_id` and `connect` returns the shape below.
  - Assertions read chain tables in Node and compare the UI against on-chain truth via the
    math mirror — exact, not approximate.
- **Mock `window.ultra` surface (what `@ultraos/wallet-sdk@0.6.1` actually calls)** — read from
  the published package's `dist/index.mjs` + typings. The Extension provider is a thin
  pass-through: each SDK method calls the same-named `window.ultra` method and returns its
  result **unchanged**, so every mock method must be `async` and resolve to the envelope
  `{ status: 'success' | 'fail' | 'error', data, message?, code? }`.
  - **Detection:** auto mode picks Extension iff `'ultra' in window` **when `new UltraWalletSDK()`
    runs** → install the mock with `page.addInitScript` (before app code).
  - `connect(params?)` → the SDK **first calls `getChainId()`** and reads `.data`; if the SDK was
    constructed with `environment: 'mainnet'|'testnet'` that chain ID must equal the public
    network's or `connect` throws *Wallet environment mismatch* (no/other `environment` = no
    check — use that for a local chain). Then `window.ultra.connect(params)` → `data`:
    `{ blockchainid: '<account>', publicKey, selectedAccount?: { accountName, permissions:
    [{ name, publicKeys: [] }] }, accounts?, network?: { name, chainId } }`.
  - `signTransaction(txOrTxs, options?)` — `txOrTxs` = one or an array of `{ contract, action,
    data, authorization: [{ actor, permission }] }`, `options` = `{ signOnly? }` → `data`:
    `{ transactionHash?, unsignedAuth?: string[], processed? }`; return `status:'fail'|'error'`
    + `message` for a chain/assert failure.
  - `signMessage(message)` → `data: { signature }`; `disconnect()` → `data: boolean`;
    `getChainId()` → `data: '<chainId>'`; `getSelectedAccount()` → `AccountInfo`;
    `getAccounts()` → `AccountInfo[]`; `getAvailableAuthorizations()` → `[{ accountName,
    permission, publicKey }]`; `getNetwork()` → `{ name, chainId, nodeUrl }`; `getNetworks()`
    → array of those; `switchNetwork(chainId)`. Stub only what your app calls.
  - **Events:** the SDK never calls `window.ultra.on`. It listens for `window.postMessage`
    messages from the same window shaped `{ type: 'EVENT', payload: { event, data } }` (`event`
    = `accountChanged` | `networkChanged` | `disconnect`) — emit those to simulate events.
    `addExtensionListener(event, id)` / `removeExtensionListener(event, id)` are **optional**
    (called via `?.` after `on()`, after a successful `connect`, and every 2 s while listeners
    exist) — omit them or return a resolved promise.
- **Provider tests:** no injected extension must construct Web SDK; injected extension must select
  Extension SDK; Web tests assert no extension-only API is called (`06` §9).
- **Real-wallet smoke:** headed extension flow and deployed Web Wallet popup flow are both required
  before claiming dual-provider production support (`06` §9).

## 7. Build & run

```bash
npm install          # VPN can block the registry
npx playwright install chromium   # once per fresh dapp/Playwright version — browser
                                  # binaries are NOT installed by npm install
# inside the devtools image (or any bare Linux/CI box) use instead:
npx playwright install --with-deps chromium   # also apt-installs the system libs; without
                                  # them launch fails: libglib-2.0.so.0: cannot open shared object
npm run dev          # Vite; VITE_NODE_URL=http://127.0.0.1:8888 for a local chain
npm test             # vitest math mirror
npm run build        # vue-tsc --noEmit + vite build
npm run qa:https     # prod build over HTTPS for real-extension QA (06 §6/§8)
```

## 8. Hosting (production)

Static SPA on **Cloudflare Pages** is the house pattern: GH Actions builds → publishes to a
`gh-pages` branch → Pages serves; the Pages project + custom domain are defined in Ultra's
private infra repo `[internal]`. Crypto-asset dapps must EU-geoblock (MiCA) via a Pages Function
`functions/_middleware.ts` (`request.cf.isEUCountry === '1'` → 403). Deploys are typically
tag-gated (`*.*.*-prod`). Details: `08` §6; full internal reference
`[internal: ultraOS-doc cloudflare-docs/WALLET_LEDGER_DEPLOYMENT.md]` (incl. the
GITHUB_TOKEN no-push-event deploy-freeze trap — a generic GitHub Actions fact worth
knowing anywhere).
