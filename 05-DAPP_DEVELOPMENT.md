# 05 — Dapp Development

**Last Updated:** 2026-07-24
**Read this to:** build the web frontend for an Ultra contract the way the shipped dapps do.
Exemplars: `/home/adam/ultra.repos/ultra-dex-dapp` (the reference), `ultra-lend-dapp`,
`ultra-farm-dapp` (Vue); `ultra-bridge-dapp` (React, production at bridge.ultra.io).
Wallet specifics live in `06`; this doc is the app around it.

---

## 1. The proven stack

Vue 3 + Vite + TypeScript, `<script setup lang="ts">`. Deliberately minimal deps —
exactly three runtime packages:

```json
"dependencies": {
  "vue": "^3.5.0",
  "@ultraos/wallet-sdk": "^0.3.2",     // signing (via the extension; no keys in the dapp)
  "@wharfkit/antelope": "^1.0.13"      // read-only chain client
}
```

**Pin `@ultraos/wallet-sdk` at `^0.3.2` or later.** 0.3.1 and earlier ship directory/
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
  "typescript": "^5.6.0"
}
```

No state library — one exported `reactive({...})` state object. React + wagmi/RainbowKit
is the bridge-dapp variant if you need EVM too.

## 2. Project structure (replicate this)

```
src/
  ultraWallet.ts    # thin wallet-sdk wrapper (singleton + isAvailable guard) — see 06 §3.1
  connection.ts     # reactive state + read APIClient + signAndPush + event sync + network switch
  config.ts         # contract names, NETWORKS[] (chainId+nodeUrl), tokens, matchNetwork()
  <domain>Client.ts # table reads + action builders for YOUR contract
  <domain>Math.ts   # math mirror of the contract (see §5)
  App.vue           # connect button, network selector, account badge
  components/*.vue  # feature panels
tests/e2e/          # Playwright + mockWallet.ts + chain.ts (Node-side signer/reads)
scripts/qa-https-server.mjs   # HTTPS server for real-extension manual QA
src/__tests__/      # vitest (math mirror)
```

Key split: `ultraWallet.ts` is a pure SDK passthrough; `connection.ts` owns app state and
sync; `<domain>Client.ts` translates UI intents into actions/reads. Config defaults:
`VITE_NODE_URL` env overrides the read endpoint (`http://127.0.0.1:8888` for local chain).

## 3. Talking to YOUR contract

Two directions, two mechanisms:

- **Writes** = wallet-signed transactions. Build plain-JSON actions (wallet-sdk shape:
  `{contract, action, data, authorization}`) and hand them to `signAndPush` (`06` §3.3).
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

  Rebuild the client whenever the wallet's network changes (`connection.ts:41-49`).

Asset strings: always the exact on-chain precision (`"1.00000000 UOS"`). Parse balances by
splitting on the space.

## 4. State & sync model

One reactive state: `{ walletAvailable, connected, account, permission, chainId, nodeUrl,
balances, syncing, busy }`. Rules (from the shipped `connection.ts`):

- The **wallet is the source of truth** for account + network; the dapp never has an
  account selector of its own.
- On mount: `initWalletSync()` (subscribe events) then `tryReconnect()`
  (`connect({onlyIfTrusted:true})`).
- On `networkChanged`: adopt the wallet's network, rebuild the read client, refresh reads.
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
    `push_transaction`), returning `{ok:false, error}` so contract asserts surface like
    the real wallet.
  - Assertions read chain tables in Node and compare the UI against on-chain truth via the
    math mirror — exact, not approximate.
- **Real-extension E2E** (heavier; `06` §8): persistent context + `--load-extension`.

## 7. Build & run

```bash
npm install          # VPN can block the registry
npx playwright install chromium   # once per fresh dapp/Playwright version — browser
                                  # binaries are NOT installed by npm install
npm run dev          # Vite; VITE_NODE_URL=http://127.0.0.1:8888 for a local chain
npm test             # vitest math mirror
npm run build        # vue-tsc --noEmit + vite build
npm run qa:https     # prod build over HTTPS for real-extension QA (06 §7.2)
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
