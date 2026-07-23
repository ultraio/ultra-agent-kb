# 08 — Testnet & Mainnet Deployment

**Last Updated:** 2026-07-23
**Read this to:** take a built, tested contract (and its dapp) to Ultra testnet, then
mainnet — including the parts Ultra gates. Sources: `docs-blockchain` (official, public),
Ultra's internal deploy runbooks `[internal]`, and on-chain verification notes.

---

## 1. The path in one screen

1. Local: build + full ultratest2 suite green (`03`/`04`).
2. Testnet: get a Pro Wallet account + UOS (faucet), buy RAM, `set contract`, QA with the
   real extension against testnet.
3. Mainnet: Ultra Pro Wallet + **KYC/KYB** + UOS → RAM → `set contract` → verify →
   (optionally) hand governance to `eosio` and treat the code as locked.
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

## 3. Account realities (both networks)

- You deploy to an **Ultra Pro Wallet** (self-custody `1aa2aa3aa4aa` account) — Ultra
  Accounts (EBA) are backend-managed and can't hold your contract. Docs are explicit:
  *"an Ultra Pro Wallet is necessary for deploying smart contracts."*
- Names are auto-generated; you cannot pick one. Dotted vanity names (`ultra.dex` style)
  are created by privileged accounts only — i.e. **granted by Ultra** (Corporate wallet /
  first-party arrangement).
- Create programmatically from an existing account:
  `cleos push action eosio newnonebact '{"creator":"<payer>","owner":{...},"active":{...},
  "max_payment":"1.00000000 UOS"}' -p <payer>` (~2 USD in UOS, oracle-priced). In the
  wallet UI: "Create an Ultra Pro Wallet".

## 4. Mainnet — what is actually gated

There is **no on-chain setcode whitelist** in `eosio.system`, but three layers make
mainnet deployment non-anonymous in practice:

1. **Ultra Pro Wallet required** (see §3).
2. **RAM is KYC-gated:** new accounts are capped at **10 KB** RAM until KYC/KYB with
   Ultra; a real contract (wasm ~50 KB+ plus tables) cannot fit. Completing KYC lifts the
   cap (unused-RAM ceiling 10 MB; bulk purchases are developer-account-only). This is the
   enforcement lever behind the docs' statement that *"Ultra requires that developers who
   wish to deploy smart contracts on Ultra platform perform a Know Your Client
   procedure."*
3. **Governance backstops:** `ultra.cntmgr` can kill-switch specific actions chain-wide,
   and `eosio.wrap` + 2/3 BPs can freeze accounts — bad actors get handled.

So the external-developer mainnet recipe is: create Pro Wallet → complete KYC/KYB with
Ultra (start at developers.ultra.io / `developers@ultra.io`) → buy UOS → buy RAM →
`cleos set contract` (identical mechanics to testnet, just a mainnet endpoint). For the
public NFT GraphQL API you separately request a `client_id` from `developers@ultra.io`.

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
