# 11 — Integrating Ultra's Core Contracts (private source, PUBLIC interface)

**Last Updated:** 2026-07-24
**Read this to:** call, read and integrate Ultra's system contracts (`eosio.token`,
`eosio.nft.ft`, `eosio`, `eosio.msig`, `eosio.oracle`, …) from your own contract or dapp —
**without any access to their source**.

> **The key fact:** the *source* of Ultra's system contracts is private, but their **interface
> is fully public and machine-readable**. Every deployed contract publishes its ABI — the
> complete list of actions, their argument types, and every table's row layout. You never need
> the C++ to integrate; you need the ABI, and it's one command away. This doc shows how to get
> it, how to read it, and the integration patterns for the two contracts you'll actually touch.

---

## 1. Discover ANY contract's interface (3 equivalent ways)

**A. From the dev image (offline, no network, no auth)** — 21 system-contract ABIs ship inside:

```bash
ls /opt/eosio.contracts/build/contracts/*/*.abi
python3 -c "import json;d=json.load(open('/opt/eosio.contracts/build/contracts/eosio.token/eosio.token.abi'));\
print([a['name'] for a in d['actions']]);print([t['name'] for t in d['tables']])"
```

**B. From a live chain with `cleos`** (mainnet/testnet — authoritative, matches what you'll
actually call in production):

```bash
cleos -u https://api.mainnet.ultra.io get abi  eosio.nft.ft
cleos -u https://api.mainnet.ultra.io get code eosio.token       # code hash
```

**C. From any HTTP client** (no cleos, no auth, no key — works from a dapp/back end):

```bash
curl -s -X POST https://api.mainnet.ultra.io/v1/chain/get_abi \
     -d '{"account_name":"eosio.nft.ft"}'
```

Verified 2026-07-24: `eosio.nft.ft` returns **59 actions / 29 tables** unauthenticated.
**Always prefer the live-chain ABI over any doc (including this one)** — the chain is the
source of truth, and the image's copy can lag a system-contract upgrade.

### Reading an ABI
- `actions[]` → `{name, type}`; the argument list is in `structs[]` under that `type`.
- `tables[]` → `{name, type, index_type, key_names}`; the row layout is `structs[type].fields`.
- Type suffixes: `?` = optional, `[]` = vector, `$` = binary-extension (a field appended in a
  later version — **may be absent on older rows**, so treat as optional in your code).
- `action_results[]` → declared return types of read-only actions.

### Turning an ABI into typed reads
`@wharfkit/antelope` consumes the ABI at runtime, so a `get_table_rows` result is already
shaped. For static TS types, generate interfaces from the ABI's `structs` (a ~30-line script)
or hand-write the few row types you touch — see the shapes in §2/§3 below.

## 2. `eosio.token` — UOS and every fungible token

**Verified interface** (from the shipped ABI):

```
action transfer(name from, name to, asset quantity, string memo)
action open(name owner, symbol symbol, name ram_payer)
action close(name owner, symbol symbol)
table  accounts  -> asset balance                                  // scope = the account name
table  stat      -> asset supply, asset max_supply, name issuer    // scope = symbol code
```
Also present: `create`, `issue`, `retire`, `burn`, `tax`/`configtax`, `configburn`,
`nettransfer`, `updatemeta` (+ `metadata`, `tokenconfig` tables). UOS is `8` decimals —
`"1.50000000 UOS"`.

**Read a balance** (dapp or contract-adjacent tooling):
```bash
cleos -u <rpc> get table eosio.token <account> accounts
# or: POST /v1/chain/get_table_rows {"code":"eosio.token","scope":"<account>","table":"accounts","json":true}
```

**Receive tokens in your contract** — the universal pattern. `eosio.token::transfer` notifies
both parties, so you react to it rather than "calling" anything:

```cpp
[[eosio::on_notify("eosio.token::transfer")]]
void on_transfer(name from, name to, const asset& quantity, const std::string& memo) {
   if (to != get_self() || from == get_self()) return;      // ignore outgoing/unrelated
   check(quantity.symbol == symbol("UOS", 8), "wrong token");
   // dispatch on memo — see 03 §4
}
```
⚠️ **Spoofed-token defense:** the `on_notify` above is bound to `eosio.token`, so only that
contract can trigger it. If you ever use a wildcard (`*::transfer`), you MUST verify
`get_first_receiver() == "eosio.token"_n` or anyone can mint a fake token and call you (`03` §6).

**Send tokens from your contract:** inline `transfer` action + the `eosio.code` permission on
your contract account (`04` §4 `addEosioCodePermission`, `08` `--add-code`).

## 3. `eosio.nft.ft` — Uniqs (Ultra's NFT standard)

Model: **factory → token** (`01` §5). A *factory* is the template/collection; *tokens* are
minted instances. Key verified shapes:

```
table factory.a -> uint64 id, name asset_manager, name asset_creator,
                   name conversion_rate_oracle_contract, asset[] chosen_rate,
                   asset minimum_resell_price, resale_share[] resale_shares,
                   uint32? mintable_window_start/end, trading_window_start/end,
                   recall_window_start/end, uint32? lockup_time,
                   name[] conditionless_receivers, uint8 stat,
                   string[] meta_uris, checksum256 meta_hash,
                   uint32? max_mintable_tokens, uint32 minted_tokens_no,
                   uint32 existing_tokens_no, uint32?$ authorized_tokens_no,
                   uint32?$ account_minting_limit
table token.b   -> uint64 id, uint64 token_factory_id, time_point_sec mint_date,
                   uint32 serial_number, int64 uos_payment, string? uri,
                   checksum256? hash, key_value_vec?$ key_values
```
Actions you'll most likely touch: `create`/`create.b` (factory), `issue`/`authmint*` (mint),
`transfer.a`, `buy`, `resell.a`/`cancelresell`, `createauct.a`/`bidauction.a`, `burn`.
There are **59 actions** — enumerate the live ABI (§1) rather than trusting any list.

**Reading a user's Uniqs:** `token.b` scoped per owner; but for anything user-facing prefer the
**indexed APIs** in `07` (dfuse GraphQL / the Ultra API) — walking `token.b` on-chain for a
whole wallet is slow and paginated. Note tables carry version suffixes (`token.a`/`token.b`,
`factory.a`) — **always confirm which one is live via the ABI**, they migrate.

⚠️ **Factory creation is Ultra-gated** (`01` §5, §8) — you generally cannot create factories on
mainnet without Ultra's involvement. Design your dapp to *consume* existing Uniqs unless you
have that agreement.

## 4. Other core contracts

| Contract | What you'd integrate | Where |
| --- | --- | --- |
| `eosio` (system) | account creation, `buyrambytes`/`refundram`, permissions, `powerup`-style resources | `01` §3, `08` §2 |
| `eosio.msig` | multisig `propose`/`approve`/`exec` for governance-owned contracts | `08` §5 |
| `eosio.oracle` | on-chain price feeds (`chosen_rate` on Uniq factories references it) | `01` §7 |
| `eosio.eba` | Ultra's EBA account model | `01` §2 |
| `eosio.kyc` | on-chain identity/KYC certificates — **not** required to deploy | `08` §4 |

Same rule for all of them: **`get abi` first**, then read the table you need.

## 5. Testing your integration locally

The dev image's local chain boots the **real** system contracts (`04`) — so your `on_notify`
handlers, `eosio.token` transfers and Uniq reads run against genuine bytecode, not mocks:

```ts
await ultraAPI.token.transferCustomTokens('alice', 'mycontract1', '5.00000000 UOS', 'buy,1');
const { rows } = await systemAPI.getTableRows('eosio.token', 'alice', 'accounts');
```
`UltraAPIv2` wraps the common ones (`.token`, `.system`, `.oracle`, `.faucet`); for anything
unwrapped, send the raw action with `transactOrThrow` using the ABI's argument names (`04` §4).

## 6. Checklist

- [ ] Pull the **live** ABI before coding; don't trust a doc (including this one) for argument
      order or table names.
- [ ] Handle `$` binary-extension fields as optional — old rows won't have them.
- [ ] Bind `on_notify` to `eosio.token::transfer` explicitly, or verify `get_first_receiver()`.
- [ ] Watch table version suffixes (`token.a` vs `token.b`).
- [ ] Grant `eosio.code` if your contract sends inline actions.
- [ ] Prefer indexed APIs (`07`) over on-chain scans for user-facing lists.
