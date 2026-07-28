# 07 — Chain Interaction & Data (endpoints, cleos, APIs)

**Last Updated:** 2026-07-23 (endpoint truth re-verified in a 2026-07-21 estate-wide
sweep `[internal]`; the tables below are self-contained)
**Read this to:** know which endpoints are actually alive, drive the chain from a CLI or
script, and read chain data for a dapp or backend.

---

## 1. Live public endpoints (the authoritative list)

### 1.1 Chain RPC (`/v1/chain/*`)

**Mainnet** — chain ID `a9c481dfbc7d9506dc7e87e9a137c931b0a9303f64fd7a1d08b8230133920097`:

| Endpoint | Notes |
| --- | --- |
| `https://api.mainnet.ultra.io` | Ultra-operated dfuse; chain API **+ dfuse GraphQL at `/graphql`** |
| `https://ultra.eosphere.io` | full API |
| `https://ultra.eosrio.io` | full API |
| `https://ultra.eosusa.io` | full API (+ Hyperion `/v2`) |
| `https://api.ultra.eossweden.org` | full API (+ Hyperion `/v2`) |
| `https://api.ultra.cryptolions.io` | ⚠️ silently IP-bans (~5 req/15 s, drops TCP SYN) — spare only, never primary |

**Testnet** — chain ID `7fc56be645bb76ab9d747b53089f132dcb7681db06f0852cfa03eaf6f7ac80e9`:

| Endpoint | `get_accounts_by_authorizers` | Notes |
| --- | --- | --- |
| `https://api.testnet.ultra.eossweden.org` | ✅ | **PRIMARY** (+ Hyperion `/v2`) |
| `https://testnet.ultra.eosrio.io` | ✅ | good |
| `https://ultra-testnet.eosphere.io` | ✅ | good |
| `https://test.ultra.eosusa.io` | ❌ (404s it) | use last (+ Hyperion `/v2`) |
| `https://api.ultra-testnet.cryptolions.io` | ✅ | ⚠️ IP-bans — spare only |

**DEAD — never use:** `api.testnet.ultra.io` (Ultra's testnet dfuse, decommissioned
2026-06-18, serves 525), `ultra.api.eosnation.io` / `ultratest.api.eosnation.io` (host up,
chain API gone), all `*.eoseoul.io` hosts. Older docs/scripts still reference these —
repoint, don't copy.

### 1.2 Explorers & EVM

- Explorers: `https://explorer.mainnet.ultra.io` / `https://explorer.testnet.ultra.io`
  (deep link: `…/account/<name>/tables?scope=<s>&tableName=<t>`).
- Ultra EVM: RPC `https://evm.ultra.eosusa.io` (mainnet) / `https://evm.test.ultra.eosusa.io`
  (testnet); explorers `evmexplorer.ultra.io` / `evmexplorer.testnet.ultra.io`.

### 1.3 GraphQL surfaces (three distinct things — don't confuse them)

1. **dfuse GraphQL (firehose-backed), mainnet only:** `https://api.mainnet.ultra.io/graphql`.
   Transaction/action history, searchable streams. No auth required in current read-only
   use (the `ledger.ultra.io` app queries it browser-direct `[internal]`). **No testnet
   equivalent exists** since
   the 2026-06-18 teardown — testnet has chain RPC + Hyperion only.
2. **Ultra NFT/Uniq public API (product):** `https://api.ultra.io/graphql` (prod) /
   `https://staging.api.ultra.io/graphql` (sandbox). OAuth2 client-credentials; request a
   `client_id` from `developers@ultra.io`. Uniq factories, Uniqs, wallet inventory, resale
   data. Source: `docs-blockchain/docs/products/nft-api/`.
3. **token-explorer `/public-graphql`** (the backend behind #2): no-auth reads, rate-limited,
   supports the `@live` directive for real-time updates (`uniq(id)`, `uniqFactories`,
   `uniqsOfWallet`, `uniqBuyOffers`, …). Architecture `[internal: ultraOS-doc
   ultra-api-stack/ULTRA_API_ARCHITECTURE.md]`.

For transaction history on testnet, use Hyperion `/v2/history/get_transaction` on eossweden
or eosusa.

## 2. cleos recipes (Ultra-flavored)

cleos ships with the Ultra toolchain (`/usr/local/bin/cleos` on this machine, v6.2.2-3.0.0).
Point it at any §1.1 endpoint with `-u`.

```bash
# wallet + keys
cleos wallet create --to-console            # or --name=X; SAVE the password
cleos wallet import --private-key 5K...     # into the unlocked wallet
cleos create key --to-console               # new keypair (5... / EOS...)

# reads
cleos -u https://ultra.eosphere.io get info
cleos -u https://ultra.eosphere.io get account <name>
cleos -u https://ultra.eosphere.io get table eosio eosio global
cleos -u https://ultra.eosphere.io get table eosio.oracle eosio.oracle finalaverage
cleos -u https://ultra.eosphere.io get currency balance eosio.token <account> UOS

# actions
cleos -u <url> push action eosio.token transfer \
  '["from","to","4.00000000 UOS","memo"]' -p from@active

# permissions
cleos -u <url> set account permission <acct> active '{"threshold":1,"keys":[{"key":"EOS...","weight":1}],"accounts":[]}' -p <acct>@owner
cleos -u <url> set account permission <contract> active --add-code   # grant eosio.code (inline actions)

# contract deploy (folder containing <name>.wasm + <name>.abi)
cleos -u <url> set contract <acct> ./build/mycontract -p <acct>@active
```

Table scope semantics: `get table <code> <scope> <table>` — scope is often an account name
(e.g. LP balances `scope=owner`) or a symbol code. All Ultra fungible tokens live on
**`eosio.token`** (single contract, per-symbol config — see `01`).

## 3. Programmatic reads (dapps/backends)

- **`@wharfkit/antelope` `APIClient`** — the shipped dapps' choice
  (`client.v1.chain.get_table_rows({code, scope, table, json:true})`,
  `get_currency_balance`, `get_info`). Snippets in `05` §3.
- **Raw HTTP:** `POST <endpoint>/v1/chain/get_table_rows` with JSON body — same shape.
- **Account discovery by key:** `POST /v1/chain/get_accounts_by_authorizers` with either
  `{"keys":["EOS…"]}` or `{"accounts":[{"actor":"...","permission":"active"}]}` — mind the
  per-endpoint capability gaps in §1.1. Requires `--enable-account-queries` on self-run
  nodes (incl. your local ultratest2 chain).
- **Bulk scanners** (reference implementations in `/home/adam/ultra.repos/blockchain-scripts`):
  `shell/account-query/account-query.sh` (all accounts via `eosio::userres`, EBA via
  `eosio.eba::ebasetup`), `shell/token-query/token-query.sh` (all tokens of a Uniq factory).

## 4. Operational habits

- Always carry a fallback endpoint list and rotate on failure — but treat cryptolions as a
  spare (its ban looks like a network outage: `connect=0.000000`).
- A "dead" endpoint has three flavors (probe accordingly): host down / host up but chain API
  removed (`Unknown Endpoint`) / silently rate-banning. Probe with `get_info` AND a real
  data call — a 200 on `/` proves nothing. Full probe recipes `[internal]`.
- Local chain default: `http://127.0.0.1:8888`, per-boot chain ID (read via `get_info`).
