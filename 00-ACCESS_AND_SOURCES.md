# 00 — Access & Sources (what's public, what's private, what to use)

**Last Updated:** 2026-07-24 (all visibility checks performed unauthenticated on this date)
**Read this to:** know which referenced repos/tools you can actually reach. This KB is
written to be usable in two modes: **internal** (Ultra dev machine with the private
checkouts) and **public** (npm + quay.io + developers.ultra.io only). Every doc marks
internal-only references with `[internal]`.

---

## 1. Fully public — use these freely

| Resource | Where | Notes |
| --- | --- | --- |
| Official developer docs | `https://developers.ultra.io` (source: `github.com/ultraio/docs-blockchain`, public) | tutorials, chain/contract reference, endpoints |
| **Dev toolchain image** | `quay.io/ultra.io/3rdparty-devtools:latest` (public; pin `:0.3.1`) | **the public way to get nodeos + cleos + keosd**; also ships CDT (`cdt-cpp`), **built system contracts at `/opt/eosio.contracts/build/contracts`**, **`@ultraos/ultratest2` preinstalled**, `/opt/templates/tipjar`, and `ultra-smoke`. **Refreshed 2026-07-24: nodeos v6.2.2-3.0.0 (Savanna, mainnet-matching), CDT 4.1.1, Node 22, ultratest2 1.0.4** — see §3. (Pre-Savanna build preserved at tag `618e324f…`.) Binaries are NOT downloadable individually. |
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

> **Prerequisites (host) — the ONLY thing you must install for the public path.**
> **Docker, with a working daemon** — verify FIRST with `docker info` (it must succeed) — plus
> network egress (to `docker pull` the ~2 GB image and to `npm install` public packages).
> **That is the entire host requirement.** If `docker info` fails you cannot compile a contract
> or run a local chain on this machine: that is an **environment limitation, not a gap in this
> KB** — say so and stop, don't fall back to a host `cdt-cpp`/`nodeos`. Everything else —
> `cdt-cpp`, `nodeos`/`cleos`/`keosd`, **Node 22 + npm**, `ultratest2`, the built system
> contracts — is inside the image, and you can even build + unit/integration-test the dapp
> inside the same container. (Your dapp's own JS deps — `@ultraos/wallet-sdk`,
> `@wharfkit/antelope`, vite/vitest — are pulled from public npm at install time: they need
> network, not a separate install.) Optional, per workflow: host Node+npm only if you develop
> the dapp *outside* the container; Chrome + the Ultra Wallet extension only for real-extension
> manual QA (the mocked-wallet path is headless and needs neither).
>
> **Windows:** run all of this from **WSL2** (Docker Desktop + the WSL2 backend), not native
> git-bash/CMD. In git-bash, MSYS rewrites the `-v` mount and `docker exec` paths (`/opt/…`
> becomes a Windows path) and the mount/compile silently target the wrong place; if you must
> use git-bash, prefix the command with `MSYS_NO_PATHCONV=1` (or double the leading slash:
> `//opt/…`). WSL2 avoids the entire class of problem — all paths are native Linux.

**Internal (Ultra dev machine):** everything in `02` §1 — native CDT/nodeos/ultratest2,
private checkouts, the DeFi exemplars. Fastest, and what the worked example used.

**Public (no private access)** — the official devtools image, **refreshed 2026-07-24** to the
mainnet-matching Savanna toolchain with everything preinstalled:

```bash
docker pull quay.io/ultra.io/3rdparty-devtools:latest     # or pin :0.3.1
docker run -dit --name ultra -p 8888:8888 -p 9876:9876 \
  -v ~/ultra_workdir:/opt/ultra_workdir quay.io/ultra.io/3rdparty-devtools:latest
# self-test (compiles the bundled Tip Jar template + runs its spec suite, expect 6/6):
docker exec ultra bash -lc ultra-smoke
# compile: cdt-cpp inside the image (or the VS Code extension)
# test:    ultratest2 -t <spec>   (preinstalled — no `npm i -g` needed)
# dapp:    npm i @ultraos/wallet-sdk @wharfkit/antelope   (all public)
```

Ships **nodeos/cleos v6.2.2-3.0.0 (Savanna, mainnet-matching)**, **CDT 4.1.1** (`cdt-cpp`),
**Node 22**, **`@ultraos/ultratest2@1.0.4` preinstalled** (so `ultratest2 -t <spec>` just
works), the **current `eosio.contracts` master build** at
`/opt/eosio.contracts/build/contracts` (adds `ultra.bridge/dex/farm/lend/rfq`), the Tip Jar
worked example at `/opt/templates/tipjar`, and `/usr/local/bin/ultra-smoke` (self-test). Build
inputs are recorded in `/opt/versions.json`. Validated 2026-07-24 — `ultra-smoke` green
(Tip Jar 6/6) on a fresh pull of the published tag. Built + published by CI
(`ultra.docker` `external.yml`), so it refreshes reproducibly.

> **Upgrade note.** Before 2026-07-24 this image shipped **nodeos v5.0.2 (pre-Savanna)**,
> CDT 4.0.1, Node 19 and only ultratest **v1**. If you need that older image it remains at the
> digest tag **`618e324fc60de62b6e65d757340c45daadbbf868`** (and `0.2.0`). The `:0.3.1` tag pins
> the current Savanna build.

**Validated 2026-07-23** (full Tip Jar flow, docker-only, host toolchain untouched), then
**re-validated the same day, from published npm, with ZERO workarounds** after the tooling
fixes below shipped: compile with the image's `cdt-cpp` ✅; `npm i -g @ultraos/ultratest2`
runs the whole spec suite ✅ (6/6 green) against the image's nodeos + its
`/opt/eosio.contracts/build/contracts`. This requires **`@ultraos/ultratest2 ≥ 1.0.4`** and
**`@ultraos/ultra-signer-lib ≥ 1.7.5`** (both published 2026-07-23); a fresh `npm i -g`
resolves both automatically.

The only per-project setup is a **spec-directory `package.json`** pointing `@ultraos/ultratest`
+ the four native plugins at the global install via `file:` paths (find the root with
`npm root -g` and substitute it for `<GLOBAL_ROOT>`):

```jsonc
{
  "name": "my-contract-tests",
  "version": "1.0.0",
  "dependencies": {
    "tsx": "4.7.1",
    "@ultraos/ultratest": "file:<GLOBAL_ROOT>/@ultraos/ultratest2/src",
    "ultratest-genesis-plugin": "file:<GLOBAL_ROOT>/@ultraos/ultratest2/src/plugins/native/genesis",
    "ultratest-system-plugin": "file:<GLOBAL_ROOT>/@ultraos/ultratest2/src/plugins/native/system",
    "ultratest-ultra-contracts-plugin": "file:<GLOBAL_ROOT>/@ultraos/ultratest2/src/plugins/native/ultraContracts",
    "ultratest-ultra-startup-plugin": "file:<GLOBAL_ROOT>/@ultraos/ultratest2/src/plugins/native/ultraStartup"
  },
  "ultratestPlugins": {
    "ultratest-genesis-plugin": "native",
    "ultratest-system-plugin": "native",
    "ultratest-ultra-contracts-plugin": "native",
    "ultratest-ultra-startup-plugin": "native"
  }
}
```

Then `npm install` in the spec dir and `ultratest2 -t <spec>`.

> **On older tooling (`ultratest2 < 1.0.4` / `ultra-signer-lib < 1.7.5`)** the public path
> needed four manual workarounds — pin `ultra-signer-lib@1.7.3` (1.7.4 rejected ultratest2's
> `http://0.0.0.0:8888` genesis endpoint), write `package.json` stubs into the bundled
> `genesis`/`system` plugin dirs, and symlink `eosio.bios.1.8.3` into the source-layout path.
> **Upgrading to the versions above is the fix** — don't re-derive the workarounds.

Noise to ignore: on first run the CLI self-fetches `tsx`; `ultra.bridge` may be skipped if it
isn't in the image's contract set. (The old `/var/run/docker.sock` startup-probe error is
silenced in ultratest2 ≥ 1.0.4.) **Stale advice, now obsolete:** older notes told you to
`npm i -g @ultraos/ultratest2` and to ignore an npm-9 error from the image's Node 19 — the
refreshed image is **Node 22 / npm 10 with ultratest2 preinstalled**, so neither applies.

**Still verify on testnet.** The image now runs the **mainnet-matching Savanna chain**
(nodeos v6.2.2-3.0.0) with current system contracts, so a green local run is a far stronger
signal than before — but it is still a local single-node chain: re-verify on testnet before
mainnet (`08`).

`@ultraos/wallet-sdk` **≥ 0.3.2** ships a bundled `dist/` + `exports` map, so it imports
cleanly in **plain Node / SSR / test runners** as well as bundlers. (On `0.3.1` and earlier a
plain-Node import fails with `ERR_UNSUPPORTED_DIR_IMPORT` — upgrade rather than work around it.)

## 4. Reading this KB without internal access

- Absolute paths under `/home/adam/...` describe the **internal reference machine** — on
  it, they are exact; elsewhere, treat them as "the private checkout of X" and use the
  public alternative above.
- References tagged `[internal]` point at private Ultra repos/docs; each is accompanied by
  enough inline context that no doc *depends* on following one.
- Everything else (chain IDs, endpoints, account model, RAM/KYC policy, wallet API,
  patterns, commands) is public information verified against public sources.
