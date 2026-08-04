# 12 — Smart Contract Security

**Last Updated:** 2026-08-04
**Read this to:** design and review an Ultra (Antelope) contract so it can't be drained,
spoofed, or silently corrupted. This is the consolidated security home — `03` teaches you to
*write* a contract, this doc teaches you not to get robbed. Every rule below is grounded in
Ultra's live system contracts (`/home/adam/spring/eosio.contracts/contracts/`) or the Spring
protocol source (`/home/adam/spring/eosio/libraries/chain/`). When a rule cites `file:line`,
that is verified behavior, not folklore.

> **The three questions to ask about every action, before writing its body:**
> 1. **Who is allowed to call this, and does my auth check name *that* account?** (§2)
> 2. **Am I about to read state that an inline action or notification is supposed to have
>    changed?** — if so, you're reading stale state (§1).
> 3. **If value flows in, did it really come to *me* from the *real* token contract?** (§3)

---

## 1. Execution ordering — the trap behind "reentrancy-like" bugs

**Inline actions and `require_recipient` notifications are queued, not synchronous.** When your
action calls `action.send()` (inline) or `require_recipient(x)`, nothing runs at the call site.
The runtime only *appends the action to a queue* (`apply_context.cpp:410`,
`_inline_actions` is a `vector<uint32_t>`) and drains that queue **after your action's WASM has
fully returned** (`exec()`, `apply_context.cpp:210-232`).

**The consequence that bites people:**

> A `multi_index` row read immediately **before** and immediately **after** an inline
> `action.send()` inside the same action returns the **identical value**. The inline handler
> that would mutate that row **has not run yet** — it runs after your function returns.

```cpp
// ❌ BROKEN — reads stale state; the inline hasn't executed
auto before = table.get(id).balance;
action(perm, other_contract, "debit"_n, args).send();  // queued, NOT run yet
auto after  = table.get(id).balance;                    // == before, ALWAYS
check(after < before, "debit applied");                 // fails every time

// ✅ CORRECT — mutate the state you depend on directly, in THIS action, before firing effects
auto row = table.get(id);
table.modify(row, same_payer, [&](auto& r){ r.balance -= amount; });  // effect first
action(perm, other_contract, "notify"_n, args).send();                // interaction after
```

**Ordering rules (all verified in `apply_context.cpp`):**

- Per action, the sequence is: **your action body → `require_recipient` notifications →
  context-free inline → regular inline**, each category strict FIFO. The whole transaction is a
  depth-first walk of that tree.
- **Atomic:** every inline/notified action shares one DB undo session with the parent
  (`transaction_context.cpp:80`) — any assert *anywhere* in the tree reverts *everything*. This
  is your safety net: a **closing assertion** in a final inline action can validate the end
  state of the entire composed flow, and the whole tx rolls back if violated.
- **Depth cap 4** (`config.hpp:86`, `default_max_inline_action_depth`) — deep inline chains hit
  `max inline action depth per transaction reached`.

**Rules that follow:**

1. **Effects before interactions.** Mutate (and invariant-check) your own state *before* any
   `transfer_out` / `require_recipient` / inline to another contract. Never leave a "I'll fix the
   state after the inline confirms" gap — there is no "after" within your action.
2. **Never branch on a value an inline was supposed to change.** If you need the post-effect
   value, compute it yourself in-line as plain data; don't re-read the table.
3. **Validate composed flows with a closing assertion**, not with mid-action reads. The
   flash-loan pattern (`03` §5) relies on exactly this: optimistic transfer out → borrower
   repays via their inline handler → a final `flashclose` asserts `repaid >= due` — the assert
   runs last and reverts the whole tx on shortfall.

> Note: `01` §8 and `03` §5.2 correctly call depth-first ordering "load-bearing" — that's the
> *upside* (it's why flash loans work). This section is the *downside*: the same queuing means
> you are blind to inline effects mid-action. Both are the same fact.

---

## 2. Authorization — gate the party who bears the cost

`require_auth(x)` hard-fails unless `x` signed the transaction. The bug is almost never a
*missing* check — it's checking the **wrong account**.

**Gate on whoever loses value or bears risk, and read that identity from state, not from an
action parameter:**

```cpp
// eosio.token.cpp:83-84 — spend requires the SPENDER, not get_self(), not a param
check( from != to, "cannot transfer to self" );
require_auth( from );

// eosio.token.cpp:38 — mint authority is DATA-DERIVED from the token's stats row
const auto& st = statstable.get( sym.raw() );
require_auth( st.issuer );          // not a hard-coded name

// eosio.token.cpp:197 — mutate an owned row → require the owner
require_auth( owner );
```

- **The classic drain bug:** an action debits account `X` (a parameter) but authorizes
  `get_self()` or a *different* account. Anyone can then pass a victim's name and drain them.
  The authorized account must be the one whose balance/asset/row is affected.
- **"Owner OR platform" idiom** (real, `eosio.token.cpp:240-242`): use `has_auth`, and pick the
  RAM payer from whoever actually signed:
  ```cpp
  check( has_auth(st.issuer) || has_auth("ultra"_n), "missing issuer or ultra authority" );
  auto ram_payer = has_auth(st.issuer) ? st.issuer : "ultra"_n;
  ```
- **Notification handlers (`on_notify`) have NO authority of their own.** `require_auth` is
  meaningless there — the transaction was authorized for the *transfer*, not for your handler.
  Authorization in a notify handler = the static binding + the receiver guards in §3. Ultra's
  handlers deliberately carry no `require_auth` (e.g. `ultra.avatar.cpp:114`).
- **Inline-only helper actions must self-guard with `get_sender()`.** An action meant to be
  called only as an inline from your own contract is still pushable directly by anyone unless you
  check the sender:
  ```cpp
  // eosio.token.cpp:207 — this action is only valid as an inline from eosio.token itself
  check( get_sender() == "eosio.token"_n, "sender must be eosio.token" );
  ```
  Without this, "internal" bookkeeping actions become a public write API.
- **Admin actions** behind a single `require_admin()` that falls back to `get_self()`; **no admin
  path may move user funds** — governance handoff instead (`08` §5).

---

## 3. Receiving value — the fake-deposit guard

`eosio.token::transfer` fires `require_recipient(from)` **and** `require_recipient(to)`
(`eosio.token.cpp:90-91`) — so your contract's handler runs whether you were the payee, the
payer, or (via a look-alike token) a complete bystander. A handler that credits an internal
balance **must** prove three things. This is the canonical Ultra order (ultra.swap
`:101-111`):

```cpp
[[eosio::on_notify("eosio.token::transfer")]]              // 1. bind to the REAL token contract
void on_transfer(name from, name to, asset quantity, string memo) {
    if (from == get_self()) return;                        // 2a. ignore self-funding / refunds
    if (to   != get_self()) return;                        // 2b. paid TO us — not a bystander notify
    check(get_first_receiver() == "eosio.token"_n,         // 3. the notifier is the real token code
          "only eosio.token accepted");
    // ...only now is `quantity` a real credit to us
}
```

- **`to == get_self()` is not optional.** Because `require_recipient(from)` also fires, a
  transfer *between two other parties* still invokes your handler with `to != self`. Crediting
  without this check lets an attacker "deposit" funds they never sent you (the forged-receipt
  class). Verified in protocol: in a notify handler `receiver` (`get_self()`) ≠ the action's
  original target (`apply_context.cpp:172`).
- **`get_first_receiver() == <token contract>` defeats fake tokens.** Anyone can deploy a
  contract that emits a `transfer` notification for a look-alike `"UOS"`/`"EOS"` symbol. Without
  this check you credit counterfeit tokens (the EOSBet 2018 class — see §4).
- **Wildcard bindings make the first-receiver check NON-NEGOTIABLE.** A static
  `on_notify("eosio.token::transfer")` binding means the runtime already guarantees the first
  receiver is `eosio.token` (this is why `ultra.discord.cpp:44-47` is safe with only from/to
  guards). But a `*::action` binding accepts *any* contract — you **must** assert the notifier
  against configured state:
  ```cpp
  // ultra.swap.cpp:50-53 — wildcard on_notify("*::issuewdata"): verify against config
  check( get_first_receiver() == itr->bridge_contract,
         "notify must come from the configured bridge contract" );
  ```
- **Validate the asset explicitly** — symbol *and* precision. Don't rely on the incidental
  `asset`-comparison throw as your only guard (an anti-pattern seen in `ultra.discord`'s
  `deposit`). Assert `quantity.symbol == expected` yourself.
- **Reject unknown memos** (revert, don't silently ignore) and **reject fee-on-transfer / taxed
  tokens** at registration — they desync explicit accounting (`03` §5.6).
- **Never validate funds by reading your own balance.** A concurrent in-flight deposit fakes it;
  this is a known CRITICAL drain class. Track dues in a transient table and assert
  `repaid >= due` (`03` §5.9). See §4.4.

---

## 4. Known attack classes on Antelope/EOSIO (with Ultra status)

Short catalogue of the historical hacks that shaped these rules. Each names the class, the
incident, and how it maps to Ultra today.

1. **Fake token / fake deposit** *(EOSBet, 2018)* — attacker deploys their own contract issuing
   a token with a look-alike symbol and "transfers" it to the dapp. A handler that reads only
   `quantity` credits counterfeit value. **→ §3: `get_first_receiver()` check.** Alive on Ultra;
   fully preventable.

2. **Forged transfer receipt / bystander notification** *(EOSBet, 2018)* — because
   `require_recipient(from)` also fires, and a malicious contract can notify you of a transfer
   you weren't the recipient of, a handler without `to == self` credits phantom deposits.
   **→ §3: `to == get_self()` check.** Alive on Ultra; fully preventable.

3. **Inline-ordering state staleness** — logic assumes an inline action's state change is visible
   within the same action; it isn't (§1). Manifests as double-spends, skipped decrements, and
   invariant checks that silently pass. **→ §1: effects-before-interactions + closing
   assertion.** Alive on Ultra; this is the "reentrancy-like" bug class here (there is no true
   synchronous reentrancy, but the blindness to queued effects produces the same defects).

4. **Balance-inference drain** — validating a deposit or loan repayment by reading the contract's
   own token balance instead of explicit accounting; a concurrent in-flight transfer fakes the
   balance. **→ §3 + `03` §5.9: explicit `due`/`repaid` table.** Alive on Ultra; designed out in
   the DeFi contracts by law, not patch.

5. **On-chain RNG predictability** *(EOSPlay / EOSDice, 2018)* — "randomness" derived from block
   time, tapos, or other on-chain-visible data is predictable and griefable; a bettor (or a
   producer) can compute or influence the outcome. **→ No secure on-chain RNG exists.** Use
   commit-reveal or an external oracle for any value that must be unpredictable. Alive on Ultra.

6. **Deferred-transaction `onerror` rollback abuse** *(EOSBet dice, 2019)* — attackers used
   deferred transactions and the `onerror` handler to observe outcomes and cancel losing bets.
   **→ ELIMINATED on Ultra: there are no deferred transactions and no native cron** (`01` §8,
   `03` §5.11). Do **not** port any pattern that assumes deferred tx — schedule off-chain
   instead.

7. **Asset / integer overflow** — unchecked `int64` asset math overflows into attacker-favorable
   amounts. **→ `03` §5.7: `unsigned __int128` intermediates, guard every multiply, bound to
   int64, round toward the protocol.** Alive on Ultra; preventable with disciplined math.

---

## 5. Security checklist (mirror in `10`)

- [ ] **Auth names the right account** — `require_auth(<party who bears the cost>)`, read from
      state, never from an unauthenticated action param; never `get_self()` for user-initiated
      spends (§2).
- [ ] **Inline-only helpers self-guard** with `check(get_sender() == <parent>, …)` (§2).
- [ ] **`on_notify` handlers carry no `require_auth`** — they have no authority; guard by binding
      + receiver checks instead (§2, §3).
- [ ] **Inbound value = three-part guard:** `from != self` → `to == self` → `get_first_receiver()
      == <token contract>`; wildcard `*::` bindings *require* the first-receiver check (§3).
- [ ] **Validate symbol + precision explicitly**; reject unknown memos; reject taxed/fee-on-
      transfer tokens at registration (§3).
- [ ] **Never read your own balance to validate funds** — explicit `due`/`repaid` accounting
      (§3, §4.4).
- [ ] **Effects before interactions** — mutate + invariant-check your state before any inline /
      `transfer_out` / `require_recipient`; never re-read a table expecting an inline's change
      (§1).
- [ ] **Composed flows end with a closing assertion** (whole tx reverts on violation) (§1).
- [ ] **No unpredictable value from on-chain data** — commit-reveal / oracle for randomness
      (§4.5).
- [ ] **No deferred-tx assumptions** — none exist on Ultra; schedule off-chain (§4.6).
- [ ] **u128 intermediates + overflow guards**, round toward the protocol (§4.7).
- [ ] **No admin path can move user funds**; pause switch halts *new* risk only; plan governance
      handoff (`08` §5).
- [ ] **Every assert has a negative-path spec** that triggers it (`04`).

Deep dives `[internal: ultraOS-doc]`: `ultra-defi/OVERALL_REVIEW_REPORT.md` (audit findings),
the five contract specs `ultra-defi/0{1..5}-*.md`, and `ultra-dex/07-ULTRA_CHAIN_PRIMITIVES.md`
(signatures / `recover_key`, RAM costs). Public readers: `09` (Tip Jar) reproduces the
load-bearing guards in a complete self-contained contract.
