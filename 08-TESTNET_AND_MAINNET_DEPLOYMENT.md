# 08 — Testnet & Mainnet Deployment

**Last Updated:** 2026-07-24
**Read this to:** take a built, tested contract (and its dapp) to Ultra testnet, then
mainnet — the account + UOS + RAM on-ramp, the exact `cleos` deploy (permissionless),
and dapp hosting. Sources: `docs-blockchain` (official, public), Ultra's internal deploy
runbooks `[internal]`, the ultra.io chain-opening announcement, and on-chain verification notes.

---

## 1. The path in one screen

1. Local: build + full ultratest2 suite green (`03`/`04`).
2. Testnet: get an account + UOS (faucet), buy RAM, `set contract`, QA with the
   real extension against testnet.
3. Mainnet: **permissionless** — any account whose key you can export (an Ultra Account/EBA, or
   a Pro Wallet) + UOS → RAM → `set contract` → verify → (optionally) hand governance to `eosio`
   and treat the code as locked.
   *(Brand-new to Ultra? §3 has the full on-ramp: register at ultra.io → extension wallet → get
   UOS (Simplex or the bridge) → export your account's private key → `cleos`.)*
4. Dapp: Cloudflare Pages (+ MiCA geoblock if crypto-asset-facing).

## 2. Testnet

- **Faucet: `https://faucet.testnet.app.ultra.io`** — two tabs:
  *Account Creator* (paste your `EOS…` public key → auto-named 12-char Pro-Wallet-style
  account) and *Token Faucet* (10 UOS per request). Generate keys with
  `cleos create key --to-console`. (If the faucet is ever down — it survived the 2026-06-18
  testnet-backend teardown, but re-verify — ask in Ultra's dev Discord or use an existing
  funded account.)
- **RAM:** `cleos -u <testnet-rpc> push action eosio buyrambytes
  '["<acct>","<acct>",200000]' -p <acct>@active` (or `buyram` with a UOS amount; or the
  Tool Kit's Transaction Builder UI).
  **Sizing:** budget **wasm size × ~2, plus table growth, plus ~64 KB headroom**. `200000`
  (~200 KB) suits a small contract; the DeFi runbooks gift **~5 MiB** because those contracts
  carry large, fast-growing tables. Check what you actually used with
  `cleos get account <acct>` (`ram_usage` vs `ram_quota`) and top up — RAM is refundable via
  `refundram`, so over-buying on testnet is cheap. (New accounts start with only Ultra's
  sponsored ~5 KB RAM, so you always buy more for a real contract — mainnet included (§4).)
- **⚠️ Prerequisite for every `cleos` command below that signs (`-p …`): a running `keosd` with
  an unlocked wallet holding the key.** Without it you get `Error 3120003: Locked wallet`. In the
  dev image `keosd` already runs in-container; on your own host start it once and import the
  private key you exported from the wallet extension (§3):

  ```bash
  # 1. start the key daemon — leave it running
  keosd --http-server-address=127.0.0.1:8899 --http-max-response-time-ms=30000 \
        --wallet-dir="$HOME/eosio-wallet" --unlock-timeout=86400 &
  # 2. create a wallet — SAVE the password it prints
  cleos --wallet-url http://127.0.0.1:8899 wallet create --to-console
  # 3. import your exported private key (the `5…`/`PVT_K1_…` WIF from §3)
  cleos --wallet-url http://127.0.0.1:8899 wallet import --private-key <WIF>
  # 4. later sessions / after the unlock-timeout: re-unlock
  cleos --wallet-url http://127.0.0.1:8899 wallet unlock --password <pw>
  ```
  (`--wallet-url` must match keosd's `--http-server-address`; omit both to use cleos's default
  keosd. More in `07` §2.)
- **Deploy:**

  ```bash
  cleos -u https://api.testnet.ultra.eossweden.org set contract <acct> \
        ./build/contracts/mycontract -p <acct>@active
  cleos -u <rpc> set account permission <acct> active --add-code   # if you send inline actions
  ```

  (Or the Ultra VS Code extension `ultraio.ultra-cpp`: `Ultra: Deploy Contract` → Testnet.)
- **No gating on testnet** — any funded account can deploy.
- Endpoints: use the §1 table in `07` (primary `api.testnet.ultra.eossweden.org`;
  `api.testnet.ultra.io` is DEAD).

## 3. Accounts: first-time onboarding + realities (both networks)

**Mainnet contract deployment is permissionless** — Ultra opened the chain (*"any developer can
create and deploy their smart contracts… developers only need to pay for RAM and deploy"*,
[ultra.io announcement](https://ultra.io/opening-ultras-blockchain-enabling-uniq-creation-smart-contract-deployment-for-everyone/)),
so there is **no Ultra authorization** required. You need three things: an account whose
private key you can export, some **UOS**, and enough **RAM**.

**Brand-new to Ultra? The zero-to-deploy-key on-ramp:**

1. **Register an Ultra account** at `https://ultra.io` (free) and **install the Ultra Wallet
   extension** + sign in (`06`). You now have an **Ultra Account (EBA)** and its keys.
2. **Get UOS** — a little, to buy RAM (and to create a Pro Wallet if you want one):
   - **Simplex** — the wallet's built-in fiat on-ramp (card/bank → UOS), where supported; or
   - **Bridge** — buy **ERC-20 UOS on Uniswap** (Ethereum) and move it over with the
     **Ultra bridge** (`bridge.ultra.io`) to your Ultra mainnet account.
3. **Choose the account you deploy to** — either works:
   - **Ultra Account (EBA)** — you can now **export its private key** from the extension and use
     it directly (simplest). Caveat: an EBA's **`owner` permission is null**, so you can't
     reset/rotate its `active` key yourself.
   - **Ultra Pro Wallet** (recommended) — "Create an Ultra Pro Wallet" in the wallet UI
     (`newnonebact`, ~2 USD in UOS). You hold **both `owner` and `active`**, so if the `active`
     key ever leaks you can reset it with `owner`. That recovery control is the security reason
     to prefer it for anything serious.
4. **Export the account's private key** from the extension (account → settings / manage keys →
   **Export Private Key** → a `5…` or `PVT_K1_…` WIF). ⚠️ **This is the bridge to `cleos`** — it's
   exactly the `<WIF>` in `cleos wallet import --private-key <WIF>` (§2). It controls the account
   **and its funds**: keep it in a **gitignored `.env`**, never commit / log / print it, never
   paste it into a dapp, and **if you drive `cleos` through an AI agent, tell the agent to never
   read, echo, or log anything from `.env`.**
5. **Buy RAM and deploy** — pure `cleos` from here (§2): unlock the wallet → `buyrambytes` →
   `set contract` → `set account permission … --add-code` → verify (§5/§7).

> **It's the same flow you already validated locally.** Mainnet deployment is the exact `cleos`
> sequence you ran against the local chain (`04`) — the only differences are `-u <mainnet-rpc>`
> and the mainnet chain id (which `cleos` reads from that endpoint). The contract, ABI, and
> commands are unchanged.

> **Testnet** is simpler still: the faucet (§2) hands out a Pro-Wallet-style account from a
> public key plus free UOS — `cleos create key --to-console`, no Simplex/bridge needed.

**Account realities:**

- Deploy target: an **Ultra Account (EBA, `aa1aa2aa3aa4`)** or an **Ultra Pro Wallet
  (`1aa2aa3aa4aa`)** — both are self-custody once you export the key; the Pro Wallet also gives
  you `owner`, so prefer it when you want key rotation/recovery.
- Names are auto-generated; you cannot pick one. Dotted vanity names (`ultra.dex` style)
  are created by privileged accounts only — i.e. **granted by Ultra** (Corporate wallet /
  first-party arrangement).
- Create a Pro Wallet programmatically from an existing account:
  `cleos push action eosio newnonebact '{"creator":"<payer>","owner":{...},"active":{...},
  "max_payment":"1.00000000 UOS"}' -p <payer>` (~2 USD in UOS, oracle-priced). In the
  wallet UI: "Create an Ultra Pro Wallet".
- **Pro Wallet accounts split owner/active keys** — the key you export for
  `cleos wallet import` is usually the **active** key. Confirm which permission
  you actually hold before signing a deploy (`cleos get account <name>` — check
  whether the `active` or `owner` key matches what you imported). Assuming you
  hold `owner` when you only hold `active` won't block the deploy itself, but
  it can block a later recovery or governance action that needs `owner`.

## 4. Mainnet — permissionless deployment

Ultra **opened smart-contract deployment to everyone**: *"Previously, deploying smart contracts
needed authorization from the Ultra team. Now, any developer can create and deploy their smart
contracts on Ultra's blockchain… developers only need to pay for RAM and deploy"*
([ultra.io](https://ultra.io/opening-ultras-blockchain-enabling-uniq-creation-smart-contract-deployment-for-everyone/)).
Concretely:

- **No on-chain setcode whitelist and no Ultra authorization.** Any account with UOS and enough
  RAM can `set contract`. (Some older developers.ultra.io tutorial pages still describe a
  pre-approval step; that predates the chain opening — the announcement and the live chain are
  authoritative.)
- **RAM is the only real cost.** New accounts start with Ultra's sponsored ~5 KB, so you buy
  what your contract needs (`buyrambytes`; refundable via `refundram`).
- **Governance backstops still exist** (they don't gate honest deploys): `ultra.cntmgr` can
  kill-switch specific actions chain-wide, and `eosio.wrap` + 2/3 BPs can freeze accounts — bad
  actors are handled after the fact, not blocked at deploy time.

**Before you broadcast to mainnet:** this step is real money and irreversible
once RAM is bought and code is set. If an AI agent prepared this deployment,
have it state — out loud or in writing — the exact account name, the wasm
sha256 it's about to deploy, and the RAM cost, and confirm all three yourself
before the signing step runs. The agent should never hold or transmit your
private key (§3 above) — you run the final signing command.

So the mainnet recipe is: get an account + export its key (§3) → buy UOS → buy RAM →
`cleos set contract` — **identical mechanics to the local chain you already validated, only the
`-u <mainnet-rpc>` endpoint (and its chain id) changes.** For the public NFT GraphQL API you
separately request a `client_id` from `developers@ultra.io`.

**First-party/system contracts** are a different class: deployed through the
`eosio.msig` producer multisig (propose → BP approvals + Ultra → exec) — e.g.
`ultra.bridge`'s owner/active are `eosio.prods@prod.major/minor`. Procedure reference:
`docs-blockchain/docs/blockchain/block-producers/maintenance/system-contract-upgrade.md`.

## 5. The shipped deploy runbook shape (from the DeFi suite §15s)

For a contract you intend to run seriously, follow the pattern the audited suite defined:

1. Verify the target account exists/is free: `cleos -u <rpc> get account <name>`.
2. Build the **exact reviewed artifact**; record `sha256sum <name>.wasm` so
   deployed == audited.
3. Create/fund the account: enough RAM for code + tables (the DeFi runbooks gift ~5 MiB
   for contract accounts).
4. `set account permission <acct> active --add-code` (inline transfers).
5. `set contract` (setcode + setabi).
6. Run any one-time `init(...)` **in the same bootstrap session** as setcode (e.g. a
   contract that binds `chain_id` into a signature domain separator — don't leave an
   init window open).
7. Point the dapp at it; operator QA with the real extension (HTTPS prod build).
8. **Governance handoff / code-lock:** switch admin to `eosio`
   (`setadmin("eosio")`-style) and/or move the account's permissions under producer-msig
   control, so future code changes require BP + Ultra approval — "as immutable as a system
   contract". Hand off all related contracts **atomically** (don't leave one on the deploy
   key while others are locked).
9. Keep a pause switch that halts NEW risk only (never blocks withdrawals/exits) — after
   handoff, pausing is itself a msig action.

## 6. Deploying the dapp

House pattern (full internal references `[internal: ultraOS-doc cloudflare-docs/
WALLET_LEDGER_DEPLOYMENT.md + devops-docs/MICA_EU_GEOBLOCK.md]`):

- **Cloudflare Pages** project defined in Ultra's private infra repo `[internal]`
  (custom domain `<name>.ultra.io`; keep custom domains proxied or Access/proxying
  silently breaks).
- CI: GitHub Actions build → `peaceiris/actions-gh-pages` publishes `dist/` to `gh-pages`
  → Pages deploys. **Trap:** a bot committing with `GITHUB_TOKEN` raises no `push` event —
  deploys silently freeze; use a `workflow_run` trigger and a drift check.
- Prod deploys are usually **tag-gated** (`*.*.*-prod` tags for the bridge) — merged ≠
  deployed.
- **MiCA:** crypto-asset dapps (DEX, bridge — not the game store) must block EU-27
  visitors at the edge (a Pages Function checking `request.cf.isEUCountry === '1'` →
  branded 403), plus a ToS restricted-jurisdictions clause. Ultra implements this at the
  app/edge layer, not via WAF rules.
- Internal tools: Cloudflare Access on `<name>.ultra.io`; never on `*.tool.ultra.io`
  (zone-lockdown conflict).

## 7. Post-deploy verification

```bash
cleos -u <rpc> get code <acct>                      # hash matches your sha256
cleos -u <rpc> get abi  <acct>                      # ABI live
cleos -u <rpc> get table <acct> <scope> <table>     # tables reachable
# push one cheap real action from a throwaway account; watch it on the explorer
```

Explorer deep-link: `https://explorer.mainnet.ultra.io/account/<acct>`. For backends,
remember irreversibility ≈ 1 s (Savanna) — index on IRREVERSIBLE.
