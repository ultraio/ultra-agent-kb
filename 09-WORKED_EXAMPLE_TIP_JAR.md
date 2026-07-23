# 09 — Worked Example: Ultra Tip Jar (contract → tests → dapp → E2E)

**Last Updated:** 2026-07-23
**Provenance:** this example was built 2026-07-23 by a **clean-room agent that knew nothing
about Ultra**, using only this KB — every gate passed on the first real attempt (contract
build ✅, 6/6 ultratest2 cases ✅ first run, vitest 9/9 ✅, `vue-tsc`+vite build ✅,
Playwright 3/3 ✅ against a real seeded chain). Use it as the template for "build me a
dapp that does X".

**The live artifacts** (build and run them — the code is the truth):

| Piece | Location |
| --- | --- |
| Contract + specs | `/home/adam/spring/eosio.contracts-tipjar` `[internal]` — worktree of the private `eosio.contracts`, branch `example/tipjar` (local commit, never pushed) |
| Dapp | `/home/adam/ultra.repos/ultra-tipjar-dapp` `[internal]` (local-only) — but §2/§4/§5 reproduce all correctness-critical code |

---

## 1. What it does

Users tip UOS by transferring to the `tipjar` contract with memo `tip,<note>`. The
contract keeps a leaderboard (`tippers.a`: lifetime total, tip count, last note) and an
**explicitly-accounted** pot (`jar.a` singleton — never a balance read). Admin-only
`withdraw(to)` pays the pot out via inline transfer. Unknown memos revert. The Vue dapp
connects the Ultra Wallet, shows the leaderboard + your balance, and signs tips.

It intentionally exercises every core pattern: memo dispatch, spoofed-token rejection,
notify-context RAM, explicit accounting, effects-before-interactions, inline transfer +
`eosio.code`, singleton, ≤12-char names, versioned tables, dev-key re-keying for E2E,
the wallet-sdk action shape, and the mocked-`window.ultra` Playwright harness.

## 2. The contract (complete, as shipped)

`contracts/tipjar/include/tipjar/tipjar.hpp`:

```cpp
#pragma once
#include <eosio/asset.hpp>
#include <eosio/eosio.hpp>
#include <eosio/singleton.hpp>
#include <string>

namespace ultra {
using namespace eosio;

class [[eosio::contract("tipjar")]] tipjar : public contract {
public:
   using contract::contract;

   [[eosio::action]]
   void withdraw( const name& to );                       // admin = the contract account

   [[eosio::on_notify("eosio.token::transfer")]]          // the tip entrypoint: memo `tip,<note>`
   void on_transfer( const name& from, const name& to, const asset& quantity,
                     const std::string& memo );

private:
   static constexpr symbol UOS_SYM{ "UOS", 8 };
   static constexpr name   TOKEN_CONTRACT = "eosio.token"_n;
   static constexpr size_t MAX_NOTE_LEN   = 256;

   struct [[eosio::table, eosio::contract("tipjar")]] tipper_v0 {
      name        account;
      asset       total;
      uint64_t    tip_count;
      std::string last_note;
      uint64_t primary_key() const { return account.value; }
      EOSLIB_SERIALIZE( tipper_v0, (account)(total)(tip_count)(last_note) )
   };
   typedef multi_index<"tippers.a"_n, tipper_v0> tippers_table;

   struct [[eosio::table, eosio::contract("tipjar")]] jar_v0 {   // explicit pot accounting
      asset pot = asset{ 0, symbol{ "UOS", 8 } };
      EOSLIB_SERIALIZE( jar_v0, (pot) )
   };
   typedef eosio::singleton<"jar.a"_n, jar_v0> jar_singleton;

   void handle_tip( const name& from, const asset& quantity, const std::string& note );
};
} // namespace ultra
```

`contracts/tipjar/src/tipjar.cpp` — note how each block is a KB rule made concrete:

```cpp
#include <tipjar/tipjar.hpp>
namespace ultra {

void tipjar::on_transfer( const name& from, const name& to, const asset& quantity,
                          const std::string& memo ) {
   if (from == get_self() || to != get_self()) return;   // 03 §4.1: inbound only

   check( get_first_receiver() == TOKEN_CONTRACT,        // 03 §4.2: spoofed-token rejection
          "tipjar: only eosio.token transfers are accepted" );
   check( quantity.symbol == UOS_SYM, "tipjar: only UOS tips are accepted" );
   check( quantity.amount > 0, "tipjar: non-positive transfer" );

   const auto comma = memo.find( ',' );                  // 03 §4.3: memo dispatch
   const std::string verb = memo.substr( 0, comma );
   if (verb == "tip" && comma != std::string::npos) {
      handle_tip( from, quantity, memo.substr( comma + 1 ) );  // note may contain commas
   } else {
      check( false, "tipjar: unrecognized memo (use 'tip,<note>')" );  // 03 §4.4: revert unknowns
   }
}

void tipjar::handle_tip( const name& from, const asset& quantity, const std::string& note ) {
   check( note.size() <= MAX_NOTE_LEN, "tipjar: note too long (max 256 chars)" );

   // 03 §5.1: in notify context we may only bill OURSELVES — get_self() pays all RAM here.
   tippers_table tippers( get_self(), get_self().value );
   auto it = tippers.find( from.value );
   if (it == tippers.end()) {
      tippers.emplace( get_self(), [&]( auto& t ) {
         t.account = from; t.total = quantity; t.tip_count = 1; t.last_note = note; });
   } else {
      tippers.modify( it, same_payer, [&]( auto& t ) {
         t.total += quantity; t.tip_count += 1; t.last_note = note; });
   }

   jar_singleton jar( get_self(), get_self().value );    // 03 §5.9: explicit accounting,
   auto s = jar.get_or_default();                        // never a self-balance read
   s.pot += quantity;
   jar.set( s, get_self() );
}

void tipjar::withdraw( const name& to ) {
   require_auth( get_self() );
   check( is_account( to ), "tipjar: 'to' account does not exist" );

   jar_singleton jar( get_self(), get_self().value );
   auto s = jar.get_or_default();
   check( s.pot.amount > 0, "tipjar: nothing to withdraw" );

   const asset payout = s.pot;                           // 03 §5.8: effects BEFORE interactions
   s.pot.amount = 0;
   jar.set( s, get_self() );

   action( permission_level{ get_self(), "active"_n },   // inline transfer → needs eosio.code
           TOKEN_CONTRACT, "transfer"_n,
           std::make_tuple( get_self(), to, payout, std::string("tipjar withdrawal") ) ).send();
}
} // namespace ultra
```

Plus `ricardian/tipjar.contracts.md.in` (a clause for `withdraw`) and the standard
`CMakeLists.txt` (`03` §1).

## 3. Build + test (the exact session)

```bash
# own worktree (03 §2)
cd /home/adam/spring/eosio.contracts
git worktree add /home/adam/spring/eosio.contracts-tipjar -b example/tipjar feature/ultra-dex-amm
cd /home/adam/spring/eosio.contracts-tipjar
#   … add contracts/tipjar/*, REGISTER in build.sh contract_list + contracts/CMakeLists.txt …
./build.sh -c ../eosio.cdt/build -C tipjar     # → build/contracts/tipjar/tipjar.{wasm,abi}

# spec suite (pipe to a file — 04 §2)
ultratest2 --contracts-dir-path=/home/adam/spring/eosio.contracts-tipjar/build/contracts \
  -t /home/adam/spring/eosio.contracts-tipjar/ultratests/tipjar/tipjar.spec.ts \
  > /tmp/tipjar-spec.log 2>&1
```

The spec (`ultratests/tipjar/tipjar.spec.ts`, 6 cases — read it in full) covers: exact
table state after tips from two users (incl. a comma-in-note case), four unknown-memo
reverts, non-admin `withdraw` revert, balance-asserted withdraw, empty-pot revert. Its
setup helper does the three-step contract bootstrap verbatim (`04` §4):

```ts
await systemAPI.createAccountFull({ name: 'tipjar', giftRam: 5*1024*1024 }, 'eosio');
await systemAPI.publishContract('tipjar', '../../build/contracts/tipjar/');
await ultraAPI.system.addEosioCodePermission('tipjar', 'active', 'tipjar');
// tips are just: ultraAPI.token.transferCustomTokens(alice, 'tipjar', '5.00000000 UOS', 'tip,gm')
```

## 4. The dapp

`/home/adam/ultra.repos/ultra-tipjar-dapp` follows the `05` §2 structure exactly
(`ultraWallet.ts` / `connection.ts` / `config.ts` / `tipjarClient.ts` / `tipjarMath.ts` +
`Leaderboard.vue` + `TipForm.vue`). The two domain calls:

```ts
// read (05 §3): leaderboard + pot straight from chain tables
const { rows } = await client.v1.chain.get_table_rows({
  code: TIPJAR, scope: TIPJAR, table: 'tippers.a', json: true, limit: 100 });

// write (06 §3.3): a tip is a wallet-signed eosio.token transfer with the routing memo
await signAndPush([{
  contract: 'eosio.token', action: 'transfer', authorization: auth(),
  data: { from: state.account, to: TIPJAR,
          quantity: toAssetString(amount),            // "5.00000000 UOS"
          memo: `tip,${note}` },
}]);
```

`tipjarMath.ts` is the (small) math mirror — raw↔display conversion, note cap, leaderboard
sort — pinned by 9 vitest cases (`05` §5: even trivial mirrors get tests, because E2E
assertions are built on them).

## 5. E2E

```bash
# terminal A — seeded keep-alive chain (RPC :8888); EXIT code=0 in the log = seeded, chain STAYS UP
cd /home/adam/spring/eosio.contracts-tipjar/ultratests/tipjar
setsid ultratest2 --contracts-dir-path=/home/adam/spring/eosio.contracts-tipjar/build/contracts \
  -t $PWD/e2e_setup.ts --keep-alive > /tmp/tipjar-chain.log 2>&1 &

# terminal B
cd /home/adam/ultra.repos/ultra-tipjar-dapp
npm install && npx playwright install chromium
npx playwright test          # 3 passed — connect+leaderboard-vs-chain, tip flow with
                             # on-chain row/pot/balance delta assertions, note-cap guard
pkill -x nodeos              # cleanup
```

`e2e_setup.ts` deploys + funds alice/bob + seeds one starter tip + **re-keys alice/bob to
the dev key last** (`04` §5/§6 — this is what lets the Playwright Node-side signer work).
`tests/e2e/mockWallet.ts` + `chain.ts` are the `06` §8 harness verbatim: a `window.ultra`
mock whose `signTransaction` really signs with the dev key and pushes, so contract asserts
surface exactly like the real wallet.

## 6. What validation taught the KB (already folded in)

The clean-room build hit exactly six friction points, all fixed in these docs:
`getTableRows` returns `TableRows<T>` (destructure `.rows`) · `npx playwright install
chromium` on fresh setups · the runner npm-installs the spec dir itself on first run ·
`transferTokens` takes whole-UOS numbers vs `transferCustomTokens` asset strings ·
keep-alive's `EXIT code=0` log line means *seeded*, not *stopped* · and this doc `09`
itself was missing. Everything else — RAM gifting, re-keying, action shapes, memo rules,
registration in `build.sh` — worked first try because docs `03`/`04`/`06` called it out.

## 7. Going to production from here

This example stops at the local chain. The remaining path is mechanical: `08` §2 (testnet:
faucet account + RAM + `set contract` + `--add-code`), §4–5 (mainnet: Pro Wallet + KYC +
the runbook shape incl. wasm sha256 + governance handoff), §6 (host the dapp on Cloudflare
Pages; MiCA check). None of it changes the code you just built — only endpoints, accounts,
and permissions.
