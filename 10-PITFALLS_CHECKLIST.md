# 10 — Pitfalls Checklist (condensed)

**Last Updated:** 2026-07-24
The one-page "why is this failing" list. Each item links to the doc with the full story.

## Contract (`03`, `04`)

- [ ] "unprivileged contract cannot increase RAM usage of another account within a notify
      context" → contract pays its own RAM in `on_notify`; gift the contract account RAM.
- [ ] Deposit handler: guard `from==self / to!=self`, validate `get_first_receiver()`,
      **revert unknown memos**, reject taxed tokens.
- [ ] Never validate funds by reading your own balance — explicit due/repaid accounting.
- [ ] Effects before interactions; closing assertion for composed flows.
- [ ] Names ≤12 chars (account/action/table); `*.a` table + `*_v0` struct versioning.
- [ ] u128 intermediates + overflow guards; floor toward the protocol.
- [ ] New contract registered in `build.sh` contract_list + `contracts/CMakeLists.txt`
      (test helpers → `ultratests/CMakeLists.txt`).
- [ ] Work in your OWN `eosio.contracts` worktree `[internal]` — DeFi exemplars are in
      `/home/adam/spring/eosio.contracts-defi` (NOT the main checkout).

## Testing (`04`)

- [ ] `ultratest2 --help` HANGS — never run it. Pipe spec output to a file.
- [ ] Always pass `--contracts-dir-path=<...>/build/contracts`.
- [ ] `requiredAccounts` have RANDOM keys — re-key via `updateauth` for external signing
      ("declares authority … but does not have signatures for it").
- [ ] `publishContract` does NOT create the account — `createAccountFull({giftRam})` first,
      then `addEosioCodePermission`.
- [ ] `assertAsyncThrow(promise, x)` — x is an error SUBSTRING.
- [ ] Stuck chain: `pkill -x nodeos`; flaky infra errors (ECONNRESET, duplicate
      transaction, feature activation) → retry.
- [ ] Oracle pushes no rate locally; `pushRatesForNoWait` works once on a FRESH oracle;
      dynamic prices need a mock oracle.
- [ ] ultratest2 action shape `{account, name, authorization, data}`.

## Wallet & dapp (`05`, `06`)

- [ ] `window.ultra` injects on HTTPS + loopback only — feature-detect; serve QA builds
      over HTTPS (`qa:https`; Vite dev-HTTPS breaks on Node 22).
- [ ] wallet-sdk action shape `{contract, action, data, authorization}` (≠ raw shape).
- [ ] Check `UltraResponse.status`, handle code 4001 (declined), check `unsignedAuth`.
- [ ] `accountChanged` payload untrusted → re-query `getSelectedAccount()`;
      `selected:null` ≠ logout; never echo `disconnect`.
- [ ] Guard switchNetwork↔networkChanged loops (`syncing` flag); rebuild the read client
      on network change; localhost chainId is per-boot (via `get_info`).
- [ ] Asset strings exact precision (`"1.00000000 UOS"`).
- [ ] Math mirror must equal contract math bit-for-bit (BigInt; vitest-enforced).
- [ ] **`bool` table fields read back as `1`/`0`, never `true`/`false`** — `row.flag === true`
      is always false. Use `Boolean(row.flag)`. (Numeric `uint64` may arrive as a string.)
- [ ] A WharfKit error's contract message is in `err.response.json.error.details[].message`,
      NOT `err.message` — `String(e.message)` makes revert-tests vacuously pass.
- [ ] `--keep-alive` chains are **stateful across runs** — mutating E2E specs must be
      idempotent or the chain re-seeded.
- [ ] Pin `@ultraos/wallet-sdk` `^0.3.2`+ (earlier can't be imported by vitest/Node/SSR).
- [ ] Local chain for wallet flows needs `--enable-account-queries` (ultratest2 default).

## Integrating core contracts (`11`)

- [ ] System-contract **source is private, ABI is public** — `cleos get abi <acct>` (or
      `/v1/chain/get_abi`) before coding; never guess argument order or table names.
- [ ] Bind `on_notify` to `eosio.token::transfer` explicitly (or verify `get_first_receiver()`)
      — a wildcard lets a fake token call you.
- [ ] Treat `$` binary-extension ABI fields as optional; watch table version suffixes
      (`token.a` vs `token.b`).

## Endpoints & deploy (`07`, `08`)

- [ ] `api.testnet.ultra.io` is DEAD; eosnation + eoseoul hosts DEAD; cryptolions IP-bans
      (never primary); `test.ultra.eosusa.io` lacks `get_accounts_by_authorizers`.
- [ ] No testnet dfuse GraphQL exists anymore — mainnet only (`api.mainnet.ultra.io/graphql`).
- [ ] Every signing `cleos` command needs an **unlocked wallet** holding the key
      (`cleos wallet create/import/unlock`) or you get `Error 3120003: Locked wallet`.
- [ ] Mainnet: Pro Wallet + KYC (RAM cap 10 KB pre-KYC) + UOS. Record wasm sha256;
      `--add-code`; init in the same bootstrap; atomic governance handoff.
- [ ] Pages deploys: tag-gated; `GITHUB_TOKEN` bot pushes fire no `push` event (silent
      deploy freeze); MiCA EU-block for crypto-asset dapps.
