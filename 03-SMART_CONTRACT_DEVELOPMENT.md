# 03 — Smart Contract Development

**Last Updated:** 2026-07-24
**Read this to:** write, structure, and build an Ultra (Antelope) C++ contract the way the
shipped production suite does. The best living exemplars are the five DeFi contracts in
`/home/adam/spring/eosio.contracts-defi/contracts/` (`ultra.dex` is the reference).

---

## 1. Project layout (production convention)

```
contracts/<name>/
  include/<name>/<name>.hpp     # tables, action prototypes, helpers
  src/<name>.cpp                # definitions
  ricardian/<name>.contracts.md.in   # one ricardian clause per action
  CMakeLists.txt
```

Per-contract `CMakeLists.txt` (exact minimal shape, from `ultra.dex`):

```cmake
cmake_minimum_required(VERSION 3.5)
project(mycontract)
find_package(cdt)
include(CDTWasmToolchain)
add_contract(mycontract mycontract ${CMAKE_CURRENT_SOURCE_DIR}/src/mycontract.cpp)
target_include_directories(mycontract PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
set_target_properties(mycontract PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
# + configure_file(...) / -R flags for ricardian if you have clauses
#
# Ricardian format: the file MUST be named <contract>.contracts.md (the .in is a CMake
# template) and live in a dir passed via `-R <dir>`. One block per action:
#
#   <h1 class="contract">actionname</h1>
#   ---
#   spec_version: "0.2.0"
#   title: Human Readable Title
#   summary: 'One-line summary of what {{nowrap actionname}} does.'
#   icon: https://example.com/icon.png#hash
#   ---
#   Free-form markdown body shown to the signer.
#
# Omitting a clause only produces a `does not have a ricardian contract` WARNING —
# compilation still succeeds, so this never blocks you.
```

Conventions (mirror `ultra.dex`): `[[eosio::contract("name")]]` / `[[eosio::action]]` /
`[[eosio::table]]` / `[[eosio::on_notify(...)]]` attributes; **versioned tables** — C++
struct `*_v0` + on-chain table name `*.a` (`config.a`, `pairs.a`) with explicit
`EOSLIB_SERIALIZE`; Doxygen on public surface; each primitive its OWN small contract
composing via inline actions, not a mega-contract. Action/table/account names ≤12 chars
base32 — pick names that fit (`removeliquid`, not `removeliquidity`).

## 2. Building

**In-repo (`build.sh`) — the canonical path.** Work in your OWN git worktree of
`eosio.contracts` (the repo is shared by multiple agents):

```bash
cd /home/adam/spring/eosio.contracts
git worktree add /home/adam/spring/eosio.contracts-<yourtask> -b <your-branch> feature/ultra-dex-amm
cd /home/adam/spring/eosio.contracts-<yourtask>
# add contracts/<name>/{include,src,CMakeLists.txt}
# REGISTER IT: add <name> to build.sh's contract_list AND contracts/CMakeLists.txt
./build.sh -c ../eosio.cdt/build -C <name>     # → build/contracts/<name>/<name>.{wasm,abi}
```

Branching from `feature/ultra-dex-amm` gives you the DeFi exemplars + ultratests wiring
in-tree. A fresh worktree has no `build/` — the first `build.sh` run creates it. System
contracts must exist in `build/contracts/` for the test chain (build all once if missing:
`./build.sh -c ../eosio.cdt/build`). Flags: `-C <name>` one contract; `-l <spring-build>`
only for native C++ tests; a new **test-helper** contract registers in
`ultratests/CMakeLists.txt` instead.

**Standalone (`cdt-cpp`) — fine for a single-file contract:**
`cdt-cpp -o hello.wasm hello.cpp` emits both `hello.wasm` and `hello.abi`
(add `-I include` as needed). This is the official-docs path (VS Code extension wraps it).

## 3. A minimal real contract

The cleanest complete example is the flash-borrower test helper
(`eosio.contracts-defi/ultratests/ultra.dex/flashborrower/src/flashborrower.cpp`) —
singleton + action + inline action + `on_notify`:

```cpp
#include <eosio/eosio.hpp>
#include <eosio/asset.hpp>
#include <eosio/singleton.hpp>
using namespace eosio;

class [[eosio::contract("mycontract")]] mycontract : public contract {
public:
  using contract::contract;

  [[eosio::action]] void doit(const name& who, uint64_t amount) {
    require_auth(who);                                   // ALWAYS gate by auth first
    // inline action (depth-first; needs eosio.code on our active permission):
    action(permission_level{get_self(), "active"_n},
           "eosio.token"_n, "transfer"_n,
           std::make_tuple(get_self(), who, asset{(int64_t)amount, symbol{"UOS",8}},
                           std::string("payout"))).send();
  }

  [[eosio::on_notify("eosio.token::transfer")]]
  void on_transfer(const name& from, const name& to, const asset& quantity,
                   const std::string& memo) {
    if (from == get_self() || to != get_self()) return;  // ignore own outflows / bystander notifies
    const name token_contract = get_first_receiver();    // WHICH token contract notified us
    // dispatch on memo (see §4) …
  }

private:
  struct [[eosio::table, eosio::contract("mycontract")]] thing_v0 {
    name     owner;
    int64_t  value;
    uint64_t primary_key() const { return owner.value; }
    EOSLIB_SERIALIZE(thing_v0, (owner)(value))
  };
  typedef multi_index<"things.a"_n, thing_v0> things_table;
};
```

Secondary index example (from `pair_v0`):
`indexed_by<"bytokens"_n, const_mem_fun<pair_v0, checksum256, &pair_v0::by_tokens>>`.
Singleton: `typedef eosio::singleton<"config.a"_n, config_v0> config_singleton;`.

**Header trivia that bites (verified):** `<eosio/eosio.hpp>` does NOT transitively pull in
everything. `current_time_point()` / block-time helpers need `#include <eosio/system.hpp>`;
the `time_point_sec` type + durations need `<eosio/time.hpp>`; `checksum256` etc. need
`<eosio/crypto.hpp>`. Symptom: *"use of undeclared identifier 'current_time_point'"* on code
that looks correct — add the header, not a workaround.

## 4. The transfer + memo-dispatch pattern (inflows)

Antelope has **no ERC-20 allowance/`transferFrom`**. Users move funds INTO your contract by
sending `eosio.token::transfer` **to** your account with a routing **memo**; your
`[[eosio::on_notify("eosio.token::transfer")]]` handler parses and dispatches. Rules from
`ultra.dex.cpp:363-399`:

1. Guard first: `if (from == get_self() || to != get_self()) return;`
2. Identify the token by **`extended_symbol{quantity.symbol, get_first_receiver()}`** —
   `get_first_receiver()` is the contract that notified you; without it a look-alike token
   contract can spoof deposits.
3. `split(memo, ',')` → route to handlers (`swap,<pair>,<min_out>,<receiver>` style).
4. **Revert on any unrecognized memo** — otherwise funds land silently and are lost.
5. Memo entrypoints never appear in the ABI action list — document them.
6. **`eosio.token` caps the memo at 256 bytes itself** (`check(memo.size() <= 256, "memo has
   more than 256 bytes")`, verified in `eosio.token.wasm`) — that revert fires in the **token
   contract, before your `on_notify` ever runs**. So the ENTIRE routing memo (verb + every arg)
   must fit in 256 bytes; a free-text field pushed near 256 trips the token contract, not your
   own length check. Budget the memo, and put user-supplied text last/short.

Outflows are named actions that end in an **inline** `eosio.token::transfer` from
`get_self()` (requires `eosio.code` on your `active` permission — the deploy step
`--add-code` / `addEosioCodePermission`).

## 5. Platform rules that WILL bite you (all verified)

1. **Notify-context RAM:** an unprivileged contract can only bill **its own** RAM inside an
   `on_notify` handler — never the user's, even though the user signed. So rows created on
   deposit are paid by `get_self()` → **fund the contract account with RAM** (local tests:
   `createAccountFull({giftRam: 5*1024*1024})`). nodeos error text: *"unprivileged contract
   cannot increase RAM usage of another account within a notify context"*. In your own
   user-authorized actions you may bill the user (`ram_payer = user`), and reclaim rows on
   full withdrawal.
2. **Depth-first inline execution:** a child action spawned inside a notification runs
   before the parent's next sibling. Flash-loan pattern depends on it: optimistic
   `transfer_out` → `flashnotify` (borrower's handler repays inline) → `flashclose`
   asserts repayment — all before the transaction commits.
3. **No sync calls:** compose contracts with inline actions + a **closing assertion** that
   validates the end state (whole tx reverts if violated).
4. **Two action JSON shapes:** raw push/ultratest2 = `{account, name, authorization,
   data}`; wallet-sdk = `{contract, action, data, authorization}`.
5. **≤12-char names** for accounts/actions/tables.
6. **Tokens:** identify by `extended_symbol`; **reject taxed tokens** (read `eosio.token`'s
   `tokenconfig` row — field order must match exactly; see `ultra.dex.hpp:171-181` +
   `is_taxed`).
7. **u128 math:** use `unsigned __int128` intermediates, guard every multiply
   (division-inverse check), bound results to int64, round in the pool's/protocol's favor
   (`ultra.dex.cpp:34-51`).
8. **Effects before interactions:** mutate state (and K-check it) BEFORE any `transfer_out`
   / `require_recipient` (`ultra.dex.cpp:429-435`).
9. **Explicit accounting over balance reads:** never validate repayment/deposits by reading
   your own token balance — a concurrent in-flight deposit fakes it. Raw-balance repayment
   checks are a known CRITICAL drain-bug class on this platform. Track dues in a
   transient table, assert `repaid >= due`.
10. **`require_recipient(from); require_recipient(to);`** in your own transfer-like actions
    lets other contracts observe you (how `ultra.farm` stakes `ultra.dex` LP transfers) —
    commit state before firing them.
11. **No deferred txs/cron** — anything scheduled needs an off-chain bot.
12. **Read-only actions** (`[[eosio::action, eosio::read_only]]`) for view helpers
    — ⚠️ they compile and register in the ABI's `action_results`, but this KB has **no
    verified recipe for *calling* one** from ultratest2 or WharfKit (it needs a
    `/v1/chain/send_read_only_transaction`-style path). Treat them as unproven here:
    for anything you must read in a test or dapp, **read the table directly** (`04` §4)
    (`getamtout` style).

## 6. Security checklist (condensed from the shipped audits)

- `require_auth` on every state-mutating action; admin actions behind a
  `require_admin()` that falls back to `get_self()`.
- Reject unknown memos; reject spoofed tokens (`get_first_receiver`).
- Slippage/deadline params on user trades (`min_out`, `deadline`).
- Pause switch that halts NEW risk only — never one that can trap or move funds.
- No admin path that can move user funds; plan governance handoff (`08` §5).
- Assert invariants at transaction close (K-invariant, solvency, conservation).
- Test negative paths: every assert needs a spec that triggers it (`04`).
- Validate exact transfer amounts where the action expects a fixed price/fee —
  don't just check `quantity.amount > 0`; a caller can send less than expected
  and still pass a naive positive-amount check.

Deep dives `[internal: ultraOS-doc]`: `ultra-defi/AGENT_CONTEXT.md` §6 (gotchas), the five
contract specs `ultra-defi/0{1..5}-*.md` (design + audit records), and
`ultra-dex/07-ULTRA_CHAIN_PRIMITIVES.md` (chain primitives, signatures/`recover_key`,
RAM costs). Public readers: `09` reproduces every load-bearing pattern in a complete,
self-contained example.
