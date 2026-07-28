# 04 — Contract Testing with ultratest2

**Last Updated:** 2026-07-24
**Read this to:** test a contract against a REAL local Ultra chain. ultratest2 boots a
native `nodeos` (Ultra Spring fork), deploys the full system-contract stack, runs your
TypeScript specs, and (optionally) keeps the chain alive for dapp E2E.
Living exemplars: `/home/adam/spring/eosio.contracts-defi/ultratests/` (52+ specs).

---

## 0. What kind of testing is this? (unit vs integration)

**ultratest2 specs are integration/behavioral tests**, not classic in-process unit tests: each
spec boots a real `nodeos`, deploys the real system contracts, and exercises your contract over
RPC. That is a *feature* — you test against genuine `eosio.token`/`eosio.nft.ft` bytecode
(`11`), real RAM/permission/authority behavior, and real serialization — but a run costs
~70–90 s of chain boot, so you write **a handful of specs with many cases inside**, not hundreds
of micro-tests.

Practical strategy that works today:

| Want to test | Use |
| --- | --- |
| Contract behavior, authority, table state, reverts | **ultratest2 spec cases** (this doc) — one spec, many named cases |
| Pure math//formula logic in isolation | Extract it into a **header-only pure function**, then either assert it through a spec case or mirror it in TS and `vitest` it (`05` §5 "math mirror") |
| Dapp/front-end logic | `vitest` (`05` §6) |

⚠️ **Native C++ unit tests (`eosio/tester.hpp`, `cdt-cc --fnative`, CMake
`add_native_executable`) do NOT work out of the box.** CDT 4.1.1 ships the header, but
compiling even a trivial `EOSIO_TEST_BEGIN` case fails with errors **inside CDT's own headers**
(verified 2026-07-24 in the public image). Don't burn time on it — put the assertion in a spec
case instead. (Ultra's internal `eosio.contracts` has its own native `unit_test` target; it is
not reproducible from the public toolchain.)

## 1. Mental model

- `ultratest2` = global npm CLI (`@ultraos/ultratest2`), TypeScript executed directly via
  `tsx` — no build step. Specs are discovered by the `*.spec.ts` suffix.
- On start it **pkills any running nodeos**, writes genesis/config from its
  `src/configurations/`, boots a genesis node on random open ports (the e2e_setup pattern
  pins RPC to `:8888`), then your plugin stack bootstraps the chain.
- Default nodeos args include `contracts-console`, CORS `*`, `delete-all-blocks`, and
  **`enable-account-queries=true`** (so `get_accounts_by_authorizers` works — needed by
  wallets; if you ever boot nodeos by hand, add it yourself).
- Only ONE chain at a time. Leftover nodeos on `:8888` → `pkill -x nodeos`.

## 2. Running specs (exact commands)

```bash
# one spec
ultratest2 --contracts-dir-path=/home/adam/spring/eosio.contracts-defi/build/contracts \
  -t /home/adam/spring/eosio.contracts-defi/ultratests/ultra.dex/swap_v0.spec.ts

# a whole directory (recurses, skips node_modules)
ultratest2 --contracts-dir-path=.../build/contracts -t .../ultratests/ultra.dex

# keep the chain alive after the spec (for a dapp / curl / manual poking; RPC :8888)
ultratest2 --contracts-dir-path=.../build/contracts -t $PWD/e2e_setup.ts --keep-alive
```

- `--contracts-dir-path` must point at the `build/contracts` holding your wasm **and** the
  system contracts. The default (`~/ultra/eosio.contracts/build/contracts`) is almost never
  right — always pass it.
- **NEVER run `ultratest2 --help` — it hangs.** Other flags: `-n` (bare chain, no tests),
  `-s` (bare chain WITH system bootstrap), `--snapshot`, `--exclude`, `--logging <level>`,
  `--create-test <path>` (scaffold).
- **Pipe output to a file** (`… > /tmp/spec.log 2>&1`); grep alone loses on-chain error
  detail.
- Specs need **no per-dir npm install by you** — but know the mechanism: the test dir's
  `package.json` links the ultratest2 checkout via relative paths (copy an existing test
  dir's `package.json` when creating a new suite; the relative depth assumes the worktree
  sits at `/home/adam/spring/<name>`), and **the runner itself npm-installs into the spec
  dir on first run**. Consequences: the first run needs registry access (VPN trap), and an
  untracked `node_modules/` + `package-lock.json` will appear inside your worktree
  (gitignored — expected, don't commit them).
  - **Public / global-install path (no source checkout):** ultratest2 is **already installed
    globally in the dev image** (≥ 1.0.4) — no `npm i -g` needed there; run it only if you're
    installing on a host. Either way the spec `package.json` links the plugins via `file:`
    paths into the global install (`file:$(npm root -g)/@ultraos/ultratest2/src/...`). Canonical
    shape in `00` §3 — validated end-to-end against the public image with zero workarounds.

## 3. Spec anatomy

A spec default-exports a class extending `UltraTest`:

```ts
import { assert, assertAsync, assertAsyncThrow } from '@ultraos/ultratest/apis/testApi';
import { UltraTest, UltraTestAPI } from '@ultraos/ultratest/interfaces/test';
import { UltraAPIv2, ultraStartup } from 'ultratest-ultra-startup-plugin';
import { ultraContracts } from 'ultratest-ultra-contracts-plugin';
import { genesis } from 'ultratest-genesis-plugin/genesis';
import { system, SystemAPI } from 'ultratest-system-plugin/system';

export default class Test extends UltraTest {
  requiredAccounts() { return ['alice', 'bob']; }          // auto-created (RANDOM keys!)
  nodeosConfigs() { return { config: { 'abi-serializer-max-time-ms': 100000 } }; }
  async onChainStart(ultra: UltraTestAPI) {                 // the standard plugin stack
    ultra.addPlugins([genesis(ultra), system(ultra), ultraContracts(ultra), await ultraStartup(ultra)]);
  }
  async tests(ultra: UltraTestAPI) {
    const ultraAPI = new UltraAPIv2(ultra);
    const systemAPI = new SystemAPI(ultra);
    return {
      'creates account, deploys, acts, asserts': async () => {
        await systemAPI.createAccountFull({ name: 'mycontract1', giftRam: 5*1024*1024 }, 'eosio');
        await systemAPI.publishContract('mycontract1', '../../build/contracts/mycontract/');
        await ultraAPI.system.addEosioCodePermission('mycontract1', 'active', 'mycontract1');
        await ultraAPI.transactOrThrow([{
          account: 'mycontract1', name: 'doit',
          authorization: [{ actor: 'alice', permission: 'active' }],
          data: { who: 'alice', amount: 5 },
        }]);
        const { rows } = await systemAPI.getTableRows('mycontract1', 'mycontract1', 'things.a');
        assert(rows.length === 1 && rows[0].value === 5, 'row written');   // returns TableRows<T> — destructure .rows
        await assertAsyncThrow(
          ultraAPI.transactOrThrow([...badAction]), 'missing authority');  // arg = error SUBSTRING
      },
    };
  }
}
```

Plugin bootstrap order matters: `genesis` → `system` (deploys eosio.bios/system/token/msig,
inits UOS) → `ultraContracts` (Ultra system contracts) → `ultraStartup` (creates
`ultra.eosio`/`ultra`/faucet accounts, seeds balances, initializes the oracle, gifts
default RAM 10240 bytes — a local-test default, distinct from mainnet's ~5 KB sponsorship).

## 4. The API surface you'll actually use

**Write path (`UltraAPIv2`):**
- `transactOrThrow(actions[], errMsg?)` — action shape `{account, name,
  authorization:[{actor,permission}], data}` (NOT the wallet-sdk shape).
- `.token`: `transferTokens(from,to,amount,memo)` — ⚠️ `amount` is a **number of whole
  UOS** (`transferTokens('ultra.eosio', acc, 1000)` sends 1,000 UOS);
  `transferCustomTokens(from,to,quantity,memo)` takes an **exact asset string**
  (`"5.00000000 UOS"`) and is the workhorse for memo-dispatch entrypoints; also
  `issueTokens`, `createTokens`, `getAccountBalance`. ⚠️ An account must already **hold UOS**
  before it can transfer — fund senders in setup (`transferTokens('ultra.eosio', acc, 1000)`,
  as the Tip Jar setup does) or the transfer reverts `overdrawn balance` (unfunded) /
  `no balance object found` (balance row never opened).
- `.system`: `addEosioCodePermission(acct, perm, code)`, `setpriv`, plus RAM/permission
  helpers (`giftram`, `buyrambytes`, `linkauth`, …). ⚠️ **`updateAuth` is NOT on `.system`** —
  that object only *calls* it internally; re-keying is below.
- **Re-keying an account to a known key** — needed to sign as that account from **outside** the
  chain (dapp E2E, an external signer), because `requiredAccounts()` accounts get **random**
  keys you never see. Symptom otherwise: *"transaction declares authority … but does not have
  signatures for it"*. Two ways; the raw action is version-proof and is what the image's own
  `templates/tipjar/spec/e2e_setup.ts` uses:
  - **Raw action (recommended)** — push `eosio::updateauth` directly, so it can't drift from the
    tool version:
    ```ts
    ultraAPI.transactOrThrow([{ account: 'eosio', name: 'updateauth',
      authorization: [{ actor: 'alice', permission: 'owner' }],
      data: { account: 'alice', permission: 'active', parent: 'owner',
              auth: { threshold: 1, keys: [{ key: DEV_PUB, weight: 1 }], accounts: [], waits: [] } } }]);
    ```
    Re-key `active` (parent `owner`) first, then `owner` (parent `''`).
  - **Helper:** `ultraAPI.updateAuth(name, permission, parent, threshold, keys, accounts)` —
    ⚠️ it lives on `ultraAPI` **directly, not `ultraAPI.system`**, and takes **positional** args
    (`threshold: number`, `keys: {key,weight}[]`, `accounts: {weight,permission}[]`), **not a
    single `auth` object**. Verified against the installed `@ultraos/ultratest2@1.0.4`; the older
    `ultraAPI.system.updateAuth(…, auth)` form is wrong and throws *"not a function"*.
  - **Simplest** when you don't need a *specific* actor: don't re-key — sign external
    transactions as **`eosio`**, which already holds the dev key (`04` §5).
- `.oracle`, `.faucet`, plus per-contract helpers in the plugin's `api/`.

**Setup (`SystemAPI`):**
- `createAccountFull({name, giftRam?, tryOpenTokenBalance?}, creator='eosio')` — **the**
  way to make accounts incl. dotted names (`eosio` may create `ultra.dex`-style names —
  Ultra's `newaccount` has no suffix rule for privileged creators).
- `publishContract(account, dirPath)` — deploys wasm+abi. **Does NOT create the account.**
  `dirPath` is resolved relative to the **spec file's own directory**; when in doubt pass an
  **absolute** path (e.g. `/opt/eosio.contracts/build/contracts/x/` or your `/work/build/x/`).
- `getTableRows<T>(code, scope, table, limit?, show_payer?, lower_bound?, index_position?,
  key_type?)`, `getTableByScope`.

**Reads (typed):** `ultra.api.api.contract('<code>').getTable<T>('<table>', '<scope>')`
(WharfKit under the hood).

> ⚠️ **`bool` fields come back as `1` / `0`, not `true` / `false`.** The chain serializes
> `bool` as a byte, and neither the JSON RPC nor WharfKit converts it. `row.open === true`
> is **always false** — write `Boolean(row.open)`, `row.open === 1`, or `!!row.open`. Same in
> dapp code. (Numeric `uint64` fields likewise may arrive as strings — see the BigInt-mirror
> note below.) This bites specs *and* UI, and is the single most common first-run red.

> ⚠️ **The `--keep-alive` chain is stateful and is never reset between runs.** A mutating
> E2E/seed suite that passes on run 1 will fail on run 2 (duplicate rows, "already voted",
> already-created accounts). Either make such specs **idempotent**, or restart the chain
> (kill + re-run the seed) between runs.

**Asserts:** `assert(expr, msg)`, `assertAsync(promise, msg)` (fails on throw OR falsy),
`assertAsyncThrow(promise, substr?)` — `substr` is a case-insensitive **substring** of the
stringified chain error, `sleep(ms)`.

**Best practice from the shipped suites:** compute expected values in a **BigInt mirror**
of the contract math and assert EXACT equality against table reads (see
`dexHelpers.ts:jsGetAmountOut` + `swap_v0.spec.ts`) — no tolerance bands.

## 5. Keys — the #1 confusion

`requiredAccounts()` accounts get **RANDOM keys** held inside the runner process. Only
`eosio` uses the well-known dev key
(priv `5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3`,
pub `EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV`).
Inside specs this is transparent (the runner signs). To sign from OUTSIDE (wallet mock,
Playwright, the real extension) you must **re-key the account to the dev key** via
`updateauth` — the `e2e_setup.ts` files do exactly this for alice/bob. Symptom otherwise:
`transaction declares authority '{"actor":"alice",...}' but does not have signatures for it`.

## 6. e2e_setup pattern (chain for dapp E2E / manual QA)

An `e2e_setup.ts` is a spec whose `tests()` **seeds** instead of asserting: deploy
contract(s), create+fund users, create pools/state, and **re-key the user accounts to the
dev key last**. Run with `--keep-alive` → a fully-seeded chain at `http://127.0.0.1:8888`
for Playwright or a human with the real extension. Exemplars:
`ultratests/ultra.dex/e2e_setup.ts` (single pillar),
`ultratests/composability/e2e_setup.ts` (all four DeFi pillars on one chain).
Launch it detached (`setsid`) if the parent shell may exit.

**Reading the keep-alive log:** the runner prints `ULTRATEST2 EXIT: code=0, reason="Tests
completed successfully"` when seeding finishes — **the chain is still alive**; that line is
your "seeded" signal, not a shutdown. When scripting, wait for it (or for
`Passed Test e2e_setup.ts`) and then probe `http://127.0.0.1:8888/v1/chain/get_info`.

## 7. The oracle in tests

The local chain initializes `eosio.oracle` (DUOS = USD per UOS, precision 8) but **pushes
no rate**. Seed one:
`await ultraAPI.oracle.pushRatesForNoWait('coingecko','2.00000000 DUOS','1.75000000 USD')`.
That helper **backdates ~115 s → works only on a FRESH oracle**; you cannot change price
mid-test with it. For dynamic prices deploy a **mock oracle** exposing the identical
`finalaverage` row layout (`ultratests/ultra.lend/mockoracle/`) — consumers read it
unchanged. Minute-SMA row: scope = `MINUTES` symbol code, key `window*1e4 + α*1e4`
(1.0000-MINUTES SMA → key `10000`).

## 8. Flake handling

Known infra flakes: `ECONNRESET`, `Error activating features`, `duplicate transaction`,
port collisions. The composability `run.sh` wraps a 6-attempt retry with
`pkill -9 -x nodeos` + `rm -rf data-genesis` between attempts — but it **hardcodes the main
checkout path**; on a worktree run `ultratest2` directly and retry yourself. Re-run before
believing a red spec that failed with one of those signatures.
