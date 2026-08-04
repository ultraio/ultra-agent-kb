# 10 — Pitfalls Checklist (condensed)

**Last Updated:** 2026-08-04
The one-page "why is this failing" list. Each item links to the doc with the full story.

## Contract (`03`, `04`)

- [ ] "unprivileged contract cannot increase RAM usage of another account within a notify
      context" → contract pays its own RAM in `on_notify`; gift the contract account RAM.
- [ ] Deposit handler: guard `from==self / to!=self`, validate `get_first_receiver()`,
      **revert unknown memos**, reject taxed tokens.
- [ ] `eosio.token` caps the memo at **256 bytes** and reverts (`memo has more than 256 bytes`)
      BEFORE your `on_notify` runs — the whole routing memo must fit (`03` §4).
- [ ] Never validate funds by reading your own balance — explicit due/repaid accounting.
- [ ] Effects before interactions; closing assertion for composed flows.
- [ ] Names ≤12 chars (account/action/table); `*.a` table + `*_v0` struct versioning.
- [ ] u128 intermediates + overflow guards; floor toward the protocol.
- [ ] `<eosio/eosio.hpp>` isn't enough: `current_time_point()` needs `<eosio/system.hpp>`,
      `time_point_sec` `<eosio/time.hpp>`, `checksum256` `<eosio/crypto.hpp>` (`03` §3).
- [ ] New contract registered in `build.sh` contract_list + `contracts/CMakeLists.txt`
      (test helpers → `ultratests/CMakeLists.txt`).
- [ ] Work in your OWN `eosio.contracts` worktree `[internal]` — DeFi exemplars are in
      `/home/adam/spring/eosio.contracts-defi` (NOT the main checkout).

## Security (`12`) — before shipping anything that moves value

- [ ] **Auth names the right account** — `require_auth(<party who bears the cost>)` read from
      state, never a user-supplied param, never `get_self()` for user-initiated spends.
- [ ] **Inline-only helpers self-guard** `check(get_sender()==<parent>)`; `on_notify` handlers
      carry **no** `require_auth` (they have no authority — guard by binding + receiver checks).
- [ ] **Inbound value = three-part guard:** `from!=self` → `to==self` → `get_first_receiver()==
      <token>`. Wildcard `*::` `on_notify` bindings *require* the first-receiver check.
- [ ] **Inline-ordering trap:** you can't see an inline's *result* mid-action — not its state
      change (read-before == read-after) and **not its return value** (no on-chain reader). Need
      data from another contract? Read its table directly. Effects before interactions; validate
      composed flows with a closing assertion + a transient lock row (reentrancy guard).
- [ ] **Signed-message contracts** (bridge/RFQ/permit): domain separator (chain+contract) +
      used-nonce/tombstone + owner epoch mass-cancel + non-throwing `k1_recover` pre-flight.
- [ ] **No on-chain RNG** (predictable — EOSPlay class); **no deferred-tx** patterns (don't exist
      on Ultra); u128 intermediates + overflow guards; reject fee-on-transfer tokens at the boundary.

## Testing (`04`)

- [ ] `ultratest2 --help` HANGS — never run it. Pipe spec output to a file.
- [ ] Always pass `--contracts-dir-path=<...>/build/contracts`.
- [ ] `requiredAccounts` have RANDOM keys — re-key for external signing ("declares authority …
      but does not have signatures for it"). Use the raw `eosio::updateauth` action, or the
      helper `ultraAPI.updateAuth(name,perm,parent,threshold,keys,accounts)` — positional args on
      `ultraAPI` **directly**, NOT `ultraAPI.system.updateAuth(…,auth)` (that throws) (`04` §4).
- [ ] `publishContract` does NOT create the account — `createAccountFull({giftRam})` first,
      then `addEosioCodePermission`.
- [ ] Fund senders before transfer tests — an unfunded account's transfer reverts `overdrawn
      balance`; `transferTokens('ultra.eosio', acc, 1000)` in setup (`04` §4).
- [ ] `assertAsyncThrow(promise, x)` — x is an error SUBSTRING.
- [ ] Stuck chain: `pkill -x nodeos`; flaky infra errors (ECONNRESET, duplicate
      transaction, feature activation) → retry.
- [ ] Oracle pushes no rate locally; `pushRatesForNoWait` works once on a FRESH oracle;
      dynamic prices need a mock oracle.
- [ ] ultratest2 action shape `{account, name, authorization, data}`.

## Wallet & dapp (`05`, `06`)

- [ ] `window.ultra` injects on **HTTPS only** in the shipped/CWS extension (the prod build
      strips loopback) — including on `localhost`; serve QA builds over HTTPS (`qa:https`; Vite
      dev-HTTPS breaks on Node 22). Only a self-built unpacked dev extension honors
      `http://localhost` (`06` §2). Feature-detect.
- [ ] `import.meta.env` needs `"types": ["vite/client"]` in tsconfig or `vue-tsc --noEmit` fails
      (`Property 'env' does not exist on type 'ImportMeta'`) — `vite build` alone won't catch it (`05` §1).
- [ ] wallet-sdk action shape `{contract, action, data, authorization}` (≠ raw shape).
- [ ] Check `UltraResponse.status`, handle code 4001 (declined), check `unsignedAuth`.
- [ ] `accountChanged` payload untrusted → re-query `getSelectedAccount()`;
      `selected:null` ≠ logout; never echo `disconnect`.
- [ ] Guard switchNetwork↔networkChanged loops (`syncing` flag); rebuild the read client
      on network change; localhost chainId is per-boot (via `get_info`).
- [ ] Asset strings exact precision (`"1.00000000 UOS"`).
- [ ] Math mirror must equal contract math bit-for-bit (BigInt; vitest-enforced).
- [ ] Scope vitest so it doesn't run Playwright `*.spec.ts` (its default `include` matches them)
      — `test.include:['src/**/*.test.ts']` or `test.exclude:['tests/e2e/**']` (`05` §6).
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
- [ ] No deploy key yet? Register at ultra.io → extension wallet → get UOS (Simplex or the
      bridge) → **export your account's private key** → `cleos wallet import` (`08` §3).
- [ ] Public RPC has **no v1 history plugin** — `cleos get transaction` fails (`3110003`); for
      tx-by-id use Hyperion `/v2` on **`api.ultra.eossweden.org`** (history primary; eosusa also,
      but rate-limits) or mainnet dfuse GraphQL, else verify via state reads. Immediate post-tx
      reads can be **stale** on load-balanced endpoints — re-query before concluding failure (`07` §4).
- [ ] Mainnet deploy is **permissionless**: any account (EBA key exportable, or a Pro Wallet)
      + UOS + RAM. Record wasm sha256;
      `--add-code`; init in the same bootstrap; atomic governance handoff.
- [ ] **"Pro Wallet" ≠ owner/active key split** — `get account` and confirm `owner`/`active`
      resolve to **different** keys before relying on active-key recovery (`08` §3).
- [ ] Pages deploys: tag-gated; `GITHUB_TOKEN` bot pushes fire no `push` event (silent
      deploy freeze); MiCA EU-block for crypto-asset dapps.
