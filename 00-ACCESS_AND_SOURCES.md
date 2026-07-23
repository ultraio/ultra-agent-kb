# 00 — Access & Sources (what's public, what's private, what to use)

**Last Updated:** 2026-07-23 (all visibility checks performed unauthenticated on this date)
**Read this to:** know which referenced repos/tools you can actually reach. This KB is
written to be usable in two modes: **internal** (Ultra dev machine with the private
checkouts) and **public** (npm + quay.io + developers.ultra.io only). Every doc marks
internal-only references with `[internal]`.

---

## 1. Fully public — use these freely

| Resource | Where | Notes |
| --- | --- | --- |
| Official developer docs | `https://developers.ultra.io` (source: `github.com/ultraio/docs-blockchain`, public) | tutorials, chain/contract reference, endpoints |
| **Dev toolchain image** | `quay.io/ultra.io/3rdparty-devtools:latest` (public) | **the public way to get nodeos + cleos + keosd + ultratest**; also ships CDT (`cdt-cpp`) and **built system contracts at `/opt/eosio.contracts/build/contracts`**. ⚠️ latest build 2025-03-04: nodeos **v5.0.2 (pre-Savanna**; mainnet is v6.2.x**)**, CDT 4.0.1, node 19, ultratest v1 only — see §3 + `PLAN-MASTER_DEV_IMAGE.md`. Binaries are NOT downloadable individually. |
| Other public images (quay.io/ultra.io) | `eosio-docker-starter`, `eosio-cdt-docker-starter`, `3rdparty-dfuse`, `firehose-antelope` | chain/CDT starters, dfuse/firehose |
| `@ultraos/ultratest2` | npm (public) | the current TS test framework (`04`) — code installs publicly |
| `@ultraos/ultratest` | npm (public) | the v1 framework bundled in the devtools image |
| `@ultraos/wallet-sdk` | npm (public) | the dapp wallet SDK (`06`) |
| `@ultraos/ultra-signer-lib` | npm (public) | signing lib (used by ultratest2/wallet internals) |
| `@wharfkit/antelope` | npm (public) | read client + tx building |
| VS Code extension | `ultraio.ultra-cpp` (marketplace) | official compile/deploy UI (wraps cdt-cpp + deploy) |
| Chain infra | RPC endpoints, explorers, faucet, `api.mainnet.ultra.io/graphql` (dfuse), `api.ultra.io/graphql` (OAuth-gated but public product) | `07` |
| Ultra Wallet extension | Chrome Web Store | the injected `window.ultra` provider |
| Upstream Antelope | `github.com/AntelopeIO/{spring,cdt,reference-contracts}` (public) | generic platform reference — but Ultra's forks differ (see `01`) |

**GCP registries: nothing public.** Ultra's GCP/GitLab registries are private CI
infrastructure. The public distribution channels are exactly: **quay.io/ultra.io (images),
npm @ultraos (packages), developers.ultra.io (docs), Chrome Web Store (wallet)**.
(quay.io/ultraio — no dot — is a separate org of infra images: kafka-connect, dkafka etc.;
some repos are public but none are developer-toolchain images.)

## 2. Private — internal access only

| Resource | Where | Public alternative |
| --- | --- | --- |
| `ultraio/eosio.contracts` | GitHub, **private** | system-contract *behavior* is documented publicly (developers.ultra.io → blockchain/contracts); compiled system contracts ship inside the devtools image; generic patterns → AntelopeIO/reference-contracts |
| `ultraio/spring` (protocol fork) | GitHub, **private** | run nodeos via the devtools image; upstream AntelopeIO/spring for generic protocol code |
| `ultraio/eosio.cdt` (CDT fork) | GitHub, **private** | CDT inside `eosio-cdt-docker-starter` / devtools image; upstream AntelopeIO/cdt |
| `ultraio/ultratest2` (repo) | GitHub, **private** | the **npm package is public** — install that |
| `ultraio/ultra-bridge-dapp`, `wallet-ledger-app` | GitHub, **private** | — (patterns summarized in `05`/`08`) |
| GitLab `ultraio/*` (web-app incl. wallet-sdk source, ultraos, terraform, helm-charts, ultraOS-doc) | GitLab, **private** | wallet-sdk npm package + typings; extension behavior documented in `06` |
| The DeFi exemplar contracts (`ultra.dex` etc.) + local dapps (`ultra-dex-dapp`, `ultra-tipjar-dapp`) | private branch / local-only checkouts | the patterns they prove are written out in `03`–`06` and the Tip Jar code is reproduced in full in `09` |

## 3. Practical bootstrap paths

**Internal (Ultra dev machine):** everything in `02` §1 — native CDT/nodeos/ultratest2,
private checkouts, the DeFi exemplars. Fastest, and what the worked example used.

**Public (no private access):**

```bash
# toolchain + local chain (nodeos/cleos/keosd/ultratest, system contracts pre-deployed):
docker pull quay.io/ultra.io/3rdparty-devtools:latest
docker run -dit --name ultra -p 8888:8888 -p 9876:9876 \
  -v ~/ultra_workdir:/opt/ultra_workdir quay.io/ultra.io/3rdparty-devtools:latest
# compile: cdt-cpp inside the image (or the VS Code extension)
# test:    ultratest inside the image; or npm i -g @ultraos/ultratest2 on the host
# dapp:    npm i @ultraos/wallet-sdk @wharfkit/antelope   (all public)
```

**Validated 2026-07-23** (full Tip Jar flow, docker-only, host toolchain untouched):
compile with the image's `cdt-cpp` ✅; ultratest2-from-npm runs the whole spec suite ✅
(6/6 green) against the image's nodeos + its `/opt/eosio.contracts/build/contracts` —
**but only after four workarounds** an agent must apply today:

1. **Pin `@ultraos/ultra-signer-lib@1.7.3`** inside the installed ultratest2
   (`cd $(npm root -g)/@ultraos/ultratest2 && npm i @ultraos/ultra-signer-lib@1.7.3`) —
   the floating `^1.6.2` resolves to **1.7.4, which rejects ultratest2's
   `http://0.0.0.0:8888` endpoint** and breaks EVERY fresh install at genesis.
2. **Plugin stubs:** the 4 `ultratest-*-plugin` packages are NOT on npm (404), but the
   published ultratest2 tarball bundles them under `src/plugins/native/*`. Two of them
   (`genesis`, `system`) lack a `package.json` — write minimal
   `{"name":"ultratest-genesis-plugin","version":"1.0.0"}`-style stubs into those dirs.
3. **Spec-dir `package.json`:** point `@ultraos/ultratest` + the 4 plugins at
   `file:$(npm root -g)/@ultraos/ultratest2/src/...` (the internal repos' relative paths
   don't exist publicly).
4. **Bios path symlink:** the genesis plugin expects
   `/opt/eosio.contracts/contracts/eosio.bios.1.8.3` (source layout); the image only has
   `build/contracts/` — `mkdir -p /opt/eosio.contracts/contracts && ln -s` the built bios
   dir into that path.

Noise to ignore: `npm i -g` on the image's node 19 prints an npm-9 error yet yields a
working install (the CLI self-fetches `tsx` on first run); a `/var/run/docker.sock` probe
error at startup is cosmetic; `ultra.bridge` is skipped (not in the image's contract set).

**Drift warning:** a green run in this image is against a **pre-Savanna v5.0.2 chain and
older system contracts** — treat it as necessary-not-sufficient and re-verify on testnet
before mainnet. `@ultraos/wallet-sdk` needs a bundler (Vite/esbuild — fine for dapps);
plain-Node scripts can't import it directly (no `exports` map — bundle first).

The permanent fix for all of the above is the refreshed all-in-one image —
**`PLAN-MASTER_DEV_IMAGE.md`**.

## 4. Reading this KB without internal access

- Absolute paths under `/home/adam/...` describe the **internal reference machine** — on
  it, they are exact; elsewhere, treat them as "the private checkout of X" and use the
  public alternative above.
- References tagged `[internal]` point at private Ultra repos/docs; each is accompanied by
  enough inline context that no doc *depends* on following one.
- Everything else (chain IDs, endpoints, account model, RAM/KYC policy, wallet API,
  patterns, commands) is public information verified against public sources.
