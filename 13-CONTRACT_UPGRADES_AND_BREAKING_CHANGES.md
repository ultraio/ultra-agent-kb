# 13 — Contract Upgrades, Table Versioning & Breaking Changes

**Last Updated:** 2026-08-04
**Read this to:** evolve a deployed Ultra (Antelope) contract's storage **without corrupting
on-chain data** — decide whether a change is breaking, add fields the safe way, and, when a
change *is* breaking, stand up a new table version and migrate the data into it while the
contract stays live. The canonical, battle-tested exemplar is Ultra's own `eosio.nft.ft`
(the Uniq contract), which has done this repeatedly (v0→v1 factory/token migration; later a
`token.c` global-index rollout). Every rule below is drawn from its shipped source and from
Ultra's written `BREAKING_CHANGE_POLICY.md`.

> **Why this is different from an EVM upgrade.** There is no proxy/`delegatecall`. Upgrading
> the *code* is trivial — you `setcode`/`setabi` a new WASM onto the same account and the
> tables persist untouched. The danger is exactly that persistence: the new code deserializes
> **old bytes**. A table row is a packed byte blob; the ABI/struct is the only schema. Change
> that schema incompatibly and every existing row is misread or un-deserializable — silent
> data corruption, not a compile error. This doc is about keeping the byte layout compatible.

---

## 1. The one rule: on-chain table layout is APPEND-ONLY

A row is serialized field-by-field, in declaration order, with no field names or tags in the
bytes. The struct (and the ABI it generates) is the *only* schema that says how to read them.
So:

**You may NEVER, on a table that already holds rows:**
- rename a field (the name is load-bearing in the ABI and in every off-chain reader),
- change a field's **type** (`uint32_t`→`uint64_t`, `string`→`name`, reorder a struct),
- change a field's **meaning/units** (e.g. reinterpreting a "trading window" as a "transfer
  window"),
- remove a field,
- insert a field anywhere except the very end.

Each of these is a **breaking change**. The old bytes no longer decode to the new struct. The
fix is never an in-place edit — it is **§4: a new versioned table + data migration**.

**The ONLY safe in-place change** is appending a new field at the end as a **binary
extension** (§3). Anything else needs a new table.

`eosio.nft.ft` states the policy in code (`nft.common.hpp:104-110`):

```cpp
/** This is the nft standard version to migrate to.
 *  This version is manually increased each time when there are breaking changes.
 *  When migrating from one version to another, the migration logic is different each time.
 *  e.g. from version 0 to 1, factory.a and token.a needs to be migrated to factory.b and token.b
 */
constexpr uint64_t migrate_to_version = 1;
```

---

## 2. The versioning convention (naming + structs)

Ultra's convention — mirror it exactly so your contract reads like the system contracts:

- **On-chain table name carries a letter suffix**: `.a` → `.b` → `.c` … (`factory.a`,
  `factory.b`; `token.a`, `token.b`, `token.c`). The table *name* is what the chain keys on,
  so a new version literally is a **new table** living beside the old one.
- **C++ struct carries a version number**: `token_v0` → `token.a`, `token_v1` → `token.b`.
  For a straight version bump of one table the struct number and the table letter track
  together offset by one (v0↔`.a`, v1↔`.b`), because version 0 is the original `.a` table.
  The exception is when you introduce a **structurally new table family** rather than bumping
  an existing one: `token.c` is `token_global_v0` — a *new* struct family (global-scoped tokens,
  a new access pattern), so it restarts at `_v0` even though the table letter has reached `.c`.
  Rule of thumb: increment the **letter** for every new on-chain table in a lineage; the struct
  `_vN` numbers its own family.
- **Explicit `EOSLIB_SERIALIZE`** on every struct — never rely on implicit member order.
- **One `typedef multi_index` per version.** Both the old and the new typedef stay in the
  header during and after a migration; you delete the old one only once every row is gone.

From `eosio.nft.ft/include/eosio.nft.ft/nft.common.hpp`:

```cpp
struct [[eosio::table, eosio::contract("eosio.nft.ft")]] token_v0 {          // -> token.a
   uint64_t       id;
   uint64_t       token_factory_id;
   time_point_sec mint_date;
   uint32_t       serial_number;
   uint64_t primary_key() const { return id; }
   EOSLIB_SERIALIZE( token_v0, (id)(token_factory_id)(mint_date)(serial_number) )
};
typedef eosio::multi_index< "token.a"_n, token_v0 > token_table;

struct [[eosio::table, eosio::contract("eosio.nft.ft")]] token_v1 {          // -> token.b
   uint64_t       id;
   uint64_t       token_factory_id;
   time_point_sec mint_date;
   uint32_t       serial_number;
   int64_t        uos_payment = 0;                                    // appended fields
   std::optional<std::string>    uri;
   std::optional<checksum256>    hash;
   binary_extension<std::optional<key_value_vec>> key_values;         // binary extension (§3)
   EOSLIB_SERIALIZE( token_v1, (id)(token_factory_id)(mint_date)(serial_number)
                              (uos_payment)(uri)(hash)(key_values) )
   // convert a token.a row into a token.b row (used by the migration path, §4):
   token_v1& operator=(const token_v0& a) { id=a.id; token_factory_id=a.token_factory_id;
      mint_date=a.mint_date; serial_number=a.serial_number; uos_payment=0; return *this; }
};
typedef eosio::multi_index< "token.b"_n, token_v1 > token_table_v1;
```

Note the **`operator=(const old_v&)`** on the new struct: the migration code (§4) does
`new_table.emplace(payer, [&](auto& n){ n = std::move(old_row); })`, so the field-by-field
conversion — including any re-meaning of a changed field — lives in one auditable place on the
new struct.

---

## 3. Non-breaking change: append a binary extension

If all you need is to *add* a field (no existing field changes), you do **not** create a new
table. Append it to the existing struct as a **binary extension** and it decodes as "absent"
on every old row.

```cpp
// appended to the LIVE token_factory_v0 / factory.a table — no new table needed:
binary_extension<std::optional<uint32_t>> authorized_tokens_no;   // nft.common.hpp:260
binary_extension<std::optional<uint32_t>> account_minting_limit;  //             :261
```

Rules for binary extensions:
- **They must be the trailing fields** of the struct, and must appear last in
  `EOSLIB_SERIALIZE`. You can add more later, but only ever after the existing ones.
- In the **ABI** they render with a `$` type suffix (see `11` §1: *"`$` = binary-extension —
  a field appended in a later version, may be absent on older rows"*). Any off-chain reader
  must treat them as optional.
- **Read them defensively.** `binary_extension<optional<T>>` is a *double* wrapper — old rows
  have the extension disengaged; even on new rows the inner optional may be null. Unwrap both
  layers before dereferencing. Real idiom (`eosio.nft.ft.cpp:2000`):
  ```cpp
  uint32_t n = (tf->authorized_tokens_no && tf->authorized_tokens_no.value())
             ? *tf->authorized_tokens_no.value() : 0;
  ```
- **A binary extension can only ever be added, never modified or removed.** Once shipped, it
  is part of the append-only layout like any other field.
- You cannot *write* a binary-extension field on a row and leave an earlier extension unset —
  serialization is positional, so engaging field N forces fields 1..N-1 to be present too.

If the change is anything other than "append a trailing optional field," it is breaking — go
to §4.

---

## 4. Breaking change: new table + data migration

When a field must change type/meaning/name, or be removed, or a fundamentally new access
pattern is needed, the shape is always:

1. **Add a new versioned table** (`thing_v1` → `thing.b`) beside the old one; keep the old
   typedef. Put the old→new conversion in the new struct's `operator=`.
2. **Bump the version constant** (`migrate_to_version`) and add a migration-state flag for the
   table (§5).
3. **Migrate the data** from old table to new — lazily on access AND in bulk via a batched
   admin action, driven by an off-chain script (§5/§6).
4. **Point all actions at the new table**, gated so the new behavior only turns on once the
   version is activated.
5. **Retire the old table** only after every row is migrated and validated (§7).

Real example of a genuine breaking change (`token_factory_v0` → `_v1`,
`nft.common.hpp` factory structs, migration in `operator=` at `:467-508`): the v0 fields
`vector<string> meta_uris` + `checksum256 meta_hash` were **removed and replaced** by distinct
scalar fields (`string factory_uri`, `checksum256 factory_hash`, `string default_token_uri`,
`optional<checksum256> default_token_hash`) — a `vector<string>`→`string` retype plus a field
split; and the oracle fields `conversion_rate_oracle_contract` / `chosen_rate` were **dropped**
entirely. A removal or a retype of any live field is not expressible as an append — hence
`factory.b`, and a migration `operator=` that splits the old `meta_uris.front()` into **both**
new URI fields. That same `operator=` also shows migration doing real *derivation*, not just
copying: v1 adds new `transfer_window_*` fields whose initial value is **computed** from the
old `trading_window_*` (only when `minimum_resell_price == 0`) — the old `trading_window_*`
fields are carried across unchanged, so this is an *addition-with-derived-default*, not a
rename. The lesson: put every field's old→new transform (retype, split, derive) in the new
struct's `operator=` where it's auditable in one place.

Ultra runs migration with one of **two proven mechanisms**. Pick by the nature of the change:

| | **Pattern A — version-gated replace** | **Pattern B — dual-write coexistence** |
|---|---|---|
| Use when | fields changed/removed/re-meaned; old table must go away | you're adding a *parallel* table (e.g. a new index/access pattern); old table stays authoritative |
| Old rows | **erased** as they're copied (`erase` then `emplace`) | **kept**; new table is a synchronized copy |
| Version constant | bumped; `migration` singleton gates behavior | not bumped; no version gate |
| Live behavior | new actions gated behind "version activated"; each touched row migrated on the fly | every mutating action **dual-writes** both tables; new table also backfilled |
| Exemplar | `factory.a`→`.b`, `token.a`→`.b` (v0→v1) | `token.b` + `token.c` (global index) |

### Pattern A — version-gated replace (the classic)

The one used for the v0→v1 breaking change. Three moving parts:

**(a) On-the-fly (lazy) migration.** Every mutating action, at its top, migrates the specific
row it's about to touch — if migration is in progress and that row still lives in the old
table. Real helpers `migrate_factory_on_the_fly` / `migrate_token_on_the_fly`
(`eosio.nft.ft.cpp:1894-1932`), called from `transfer`, `burn`, `resell`, `buy`, … So a hot
row is migrated the first time anyone touches it; nobody waits for the bulk pass.

**(b) Batched bulk migration action.** An admin action drains the old table `total_no` rows
per call — bounded because a single transaction has a CPU limit and the table can hold
millions of rows. `mgrfactories(total_no)` (`eosio.nft.ft.cpp:1791`):

```cpp
for (auto i = factory_a_table.begin(); i != factory_a_table.end(); ) {
   token_factory_v0 old = *i;
   i = factory_a_table.erase(i);                                   // erase old…
   factory_b_table.emplace(ram_payer, [&](auto& b){ b = std::move(old); });  // …write new
   if (i == factory_a_table.end()) { _migration.set_table_migration_done(factory_a_migration_done); break; }
   else if (--total_no == 0) break;                                // stop this batch
}
```

Note **erase-old → write-new** in the same iteration, and the done-flag is set only when the
iterator reaches `end()`. For **per-owner-scoped** tables (`token.a` is scoped by owner) the
action takes a *vector of owner accounts* and drains each scope —
`migrate_tokens(owners, total_no)` / action `mgrnfts` (`:1817`) — because the contract cannot
enumerate scopes on-chain; the off-chain driver supplies the owner list (§6).

**(c) Version gate on every action.** New-version behavior is asserted behind activation:
```cpp
ASSERTION_CHECK(_migration.version_activated(1), nft_v1_is_not_activated_yet);
```
and `create()` re-routes an old-style call to `create_v1` while migration runs
(`eosio.nft.ft.cpp:31-69`). Activation is a one-shot admin action `activers()` (`:1780`) that
sets `active_nft_version = migrate_to_version` and asserts it isn't already active (no
double-activation).

### Pattern B — dual-write coexistence + idempotent backfill

Used to add the global-scope `token.c` (`token_global_v0`) index beside the per-owner
`token.b`, so the contract can query tokens by factory/owner globally. **No version bump, no
gate** — instead:

- **Every mutating action updates both tables.** Helpers `emplace_token_c` /
  `update_token_c_owner` / `update_token_c_metadata` / `erase_token_c`
  (`eosio.nft.ft.cpp:2160-2251`) are called from issue/transfer/buy/burn/setmeta/… The design
  comment: *"Coexists with token.b — both updated together … they match at any time."*
- **The updater self-heals.** `update_token_c_owner` (`:2189`): if the token isn't in
  `token.c` yet (backfill still running), it reads the row from `token.b` and creates the
  `token.c` entry on the spot — so a live edit during backfill can't create a gap.
- **Backfill is batched AND idempotent.** `duptokenc(owners, max_tokens, start_id)`
  (`:2263`) copies rows into `token.c`, **skipping any id already present**
  (`if (token_c.find(id) == token_c.end()) token_c.emplace(...)`). `max_tokens` bounds CPU;
  `start_id` resumes mid-owner. Because it copies (not erases) and skips duplicates, it is
  safe to re-run and safe to run *concurrently with live mints/transfers/burns* — the test
  harness proves convergence under exactly that load.

**When to prefer B:** you're not changing existing fields, you're adding a parallel structure.
It avoids a migration window entirely (the contract is never gated), at the cost of every
action paying to keep two tables in sync forever (or until you retire the old one).

---

## 5. Migration state: the `migration` singleton

Pattern A needs on-chain bookkeeping of *what's been activated and which tables are drained*.
`eosio.nft.ft` uses one singleton with a bitmask of per-table "done" flags
(`nft.common.hpp:156-184`):

```cpp
static constexpr uint64_t factory_a_migration_done = 0x0000'0000'0000'0001;
static constexpr uint64_t token_a_migration_done   = 0x0000'0000'0000'0002;
// add a new flag per table each time you migrate b->c, c->d, ...

struct [[eosio::table("migration"), eosio::contract("eosio.nft.ft")]] migration {
   uint64_t active_nft_version   = 0;
   uint64_t table_migration_stats = 0;                  // bitmask of *_migration_done
   EOSLIB_SERIALIZE( migration, (active_nft_version)(table_migration_stats) )

   bool version_activated(uint64_t v) const { return active_nft_version >= v; }
   bool migration_started()          const { return active_nft_version == migrate_to_version; }
   bool check_table_migration_done(uint64_t f) const { return table_migration_stats & f; }
   void set_table_migration_done(uint64_t f) {
      ASSERTION_CHECK(active_nft_version == migrate_to_version, must_activate_the_version_first);
      table_migration_stats |= f;
   }
};
typedef eosio::singleton<"migration"_n, migration> migration_singleton;
```

Loaded once in the constructor, written back in the destructor only if dirty
(`eosio.nft.ft.cpp:14-26`) — cheap for actions that don't touch it. Because per-owner-scoped
tables can't be fully enumerated on-chain, their done-flag is set **manually** by an admin
action (`setnftmgrflg`, `:1847`) once the off-chain driver confirms every scope is drained —
not auto-set by the bulk loop.

---

## 6. The off-chain migration driver

Bulk migration is *driven* off-chain — a contract can't enumerate table scopes on-chain, can't
loop over millions of rows in one tx, and can't schedule itself (no deferred tx / cron on
Ultra; see `03` §5). A Node.js/wharfkit script does the orchestration. Ultra ships two
(`eosio.nft.ft_continuous_migration/continuous_migration.js` for a→b;
`eosio.nft.ft_token_c_migration/duptokenc.js` + `validate_tokenc.js` for b→c). Its job:

1. **Enumerate scopes/owners off-chain** — `/v1/chain/get_table_by_scope` on the old table,
   or the indexer/GraphQL (`07`) — since the on-chain action needs the owner list handed in.
2. **Call the batched action in a loop**, packing up to N owners/rows per tx (prod used ~50,
   local ~100), pacing to ~one tx per block. Too-large a batch trips the CPU limit; too small
   wastes blocks.
3. **Bundle a nonce action** (e.g. `ultra.tools::nonce`) into each tx so two otherwise-identical
   migration txs don't collide as duplicate transactions.
4. **Treat "already migrated" as normal terminal state**, not an error — the loop and the
   on-the-fly path race, so a row may already be gone.
5. **Validate at the end**: count check (old table count 0 / new table count equals expected)
   **and** a field-by-field spot compare of migrated rows. Only then flip the done-flag (§5)
   and/or disable the migration actions (§7).

Grant the driver a **dedicated limited permission** (Ultra used `ultra.nft.ft@migration` with
only the migration actions `linkauth`'d to it) rather than the account's `active` key.

---

## 7. Rollout, gating & rollback (the deployment playbook)

Ultra's written `BREAKING_CHANGE_POLICY.md` generalizes the above into a rollout sequence.
Distilled and adapted for a standalone Ultra contract:

**Development**
- New code supports **both** old and new tables/actions simultaneously; the new path is
  **disabled at entry** (gated behind a version flag / `is_action_executable`), old path stays
  live. *"Never remove V1 support until V2 is fully validated."*
- Any off-chain consumers (indexers, your dapp) learn to read **both** versions.

**Deploy**
- `setcode`/`setabi` the new WASM (old path active, new path gated). Verify old behavior is
  byte-for-byte unchanged first.
- Keep the dapp on the old flow during the window.

**Activate** (short, coordinated)
- Post a maintenance notice if the switchover pauses anything.
- Flip the gate: `activers()` (activate the new version) / feature flag → new path on, old
  path off.
- Run the off-chain bulk migration (§6) to drain remaining rows (on-the-fly already handled
  hot rows).
- Upgrade the dapp to the new flow, clear the notice.

**Verify & retire**
- Validate (count + field compare); monitor.
- Set the table done-flags; **disable the now-dead migration actions**. Ultra does this
  without a redeploy via an external `disabled_action_table` checked by `is_action_executable()`
  (`nft.common.cpp:481-487`) — every migration action starts with
  `ASSERTION_CHECK(is_action_executable(...), action_is_currently_disabled)`. That's the
  on-chain equivalent of the policy's "disable V1" flag flip, and it means a migration action
  can be switched off the moment it's done, then the old table/typedef removed in a later
  release.

**Rollback** — because both paths coexist until you retire the old one, rollback is *"disable
the new path, re-enable the old,"* not a data restore. Keep it that way: don't delete old rows
in Pattern A faster than you can prove the new ones are correct, and never remove old-version
support until the new version is fully validated.

---

## 8. Decision guide

```
Changing a deployed table?
│
├─ Only ADDING a field, nothing else changes?
│     └─ Append `binary_extension<optional<T>>` at the end. Read it defensively. DONE. (§3)
│
├─ Adding a whole new PARALLEL table (new index/access pattern), existing fields unchanged?
│     └─ Pattern B: new `thing.c`, dual-write in every action, idempotent batched backfill,
│        self-healing updater. No version bump. (§4B)
│
└─ Any existing field changes type / meaning / name, or is removed?  → BREAKING
      └─ Pattern A: new `thing.b` + `_v1` struct with `operator=`, bump `migrate_to_version`,
         `migration` singleton + done-flags, on-the-fly + batched bulk migration, off-chain
         driver, version-gated actions, retire old table last. (§4A, §5, §6, §7)
```

## 9. Checklist

- [ ] Decide breaking vs non-breaking by the §1 rule (any rename/retype/re-mean/remove/insert
      = breaking). When unsure, it's breaking.
- [ ] Non-breaking add → **binary extension, trailing, optional, defensively read**; never
      touch an existing field.
- [ ] Breaking → new `_vN` struct + new `.<letter>` table, **keep the old typedef**, put
      old→new conversion in `operator=`.
- [ ] Bump the version constant; add a per-table migration done-flag; carry a `migration`
      singleton.
- [ ] Migrate **both** lazily (on-the-fly per touched row) **and** in bulk (batched admin
      action, N rows/scopes per call).
- [ ] Off-chain driver: enumerate scopes, batch, nonce each tx, tolerate "already done",
      validate (count + field compare). Run it under a dedicated limited permission.
- [ ] Gate new-version behavior behind activation; keep old path working until validated.
- [ ] Make migration actions disable-able (external flag) and disable them when done.
- [ ] Retire the old table/typedef only after every row is migrated and verified; keep
      rollback = "flip the flag," not "restore data."
- [ ] Pull the **live ABI** before and after (`11` §1) and diff it — confirm no existing
      field's name/type changed and new fields show the `$` extension suffix.

---

**Primary sources (all under `/home/adam/spring/eosio.contracts/`):**
- Versioning + `migration` singleton + version constant:
  `contracts/eosio.nft.ft/include/eosio.nft.ft/nft.common.hpp:104-184` (structs `:227-603`).
- Pattern A (bulk + on-the-fly + activation): `contracts/eosio.nft.ft/src/eosio.nft.ft.cpp:1780-1932`.
- Pattern B (`token.c` dual-write + `duptokenc` backfill): `…/eosio.nft.ft.cpp:2160-2307`.
- Action disable gate: `contracts/eosio.nft.ft/src/nft.common.cpp:481-487`.
- Off-chain drivers: `eosio.nft.ft_continuous_migration/continuous_migration.js`,
  `eosio.nft.ft_token_c_migration/{README.md,duptokenc.js,validate_tokenc.js}`.
- Written policy `[internal]`: `docs/ultra.bridge/breaking_change_management/BREAKING_CHANGE_POLICY.md`.

Related KB docs: `03` §1 (versioned-table convention, `_v0`/`.a`), `11` §1/§6 (reading a live
ABI, the `$` binary-extension suffix, watching table version suffixes), `08` (deploy /
`setcode` / msig-governed upgrades).
