# 00 — Access & Sources (what's public, what's private, what to use)

**Last Updated:** 2026-09-25 (image versions re-verified against `0.4.1-ubuntu24`/`-ubuntu22`; visibility checks performed unauthenticated 2026-07-24)
**Read this to:** know which referenced repos/tools you can actually reach. This KB is
written to be usable in two modes: **internal** (Ultra dev machine with the private
checkouts) and **public** (npm + quay.io + developers.ultra.io only). Every doc marks
internal-only references with `[internal]`.

---

## 1. Fully public — use these freely

| Resource | Where | Notes |
| --- | --- | --- |
| Official developer docs | `https://developers.ultra.io` (source: `github.com/ultraio/docs-blockchain`, public) | tutorials, chain/contract reference, endpoints |
| **Dev toolchain image** | `quay.io/ultra.io/3rdparty-devtools:0.4.1-ubuntu24` (public; **pin this tag** — Ubuntu 22.04 twin: `:0.4.1-ubuntu22`; `latest` = `0.4.1-ubuntu24` since 2026-09-25, see §3) | **the public way to get nodeos + cleos + keosd**; also ships CDT (`cdt-cpp`), **built system contracts at `/opt/eosio.contracts/build/contracts`**, **`@ultraos/ultratest2` preinstalled**, `/opt/templates/tipjar`, and `ultra-smoke`. **0.4.1: nodeos v6.2.2-3.0.1 (Savanna), CDT 4.1.1, eosio.contracts 5.2.1, Node 22.11, ultratest2 1.0.6** — see §3. (Pre-Savanna build preserved at tag `618e324f…`.) Binaries are NOT downloadable individually. |
| Other public images (quay.io/ultra.io) | `eosio-docker-starter`, `eosio-cdt-docker-starter`, `3rdparty-dfuse`, `firehose-antelope` | chain/CDT starters, dfuse/firehose |
| `@ultraos/ultratest2` | npm (public) | the current TS test framework (`04`) — code installs publicly |
| `@ultraio/ultratest` (v1) | devtools image only (`/opt/ultratest`, package 1.1.0) — **not on npm** | the legacy v1 framework; use ultratest2. Not the same thing as npm's `@ultraos/ultratest` (an unrelated 0.0.x package) or the `@ultraos/ultratest` alias a spec-dir `package.json` maps to ultratest2's `src` (§3) |
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

**Public (no private access)** — the official devtools image, **0.4.1 (2026-09)**: the
Savanna toolchain with everything preinstalled:

```bash
docker pull quay.io/ultra.io/3rdparty-devtools:0.4.1-ubuntu24     # Ubuntu 22.04: :0.4.1-ubuntu22
# one-shot self-test, no container left behind (compiles the bundled Tip Jar template +
# runs its spec suite, expect 6/6 and "== ultra-smoke: PASS =="):
docker run --rm quay.io/ultra.io/3rdparty-devtools:0.4.1-ubuntu24 -c ultra-smoke
# long-lived dev container; 8888 = nodeos's default HTTP/RPC port (the one ultratest2 uses)
docker run -dit --name ultra -p 8888:8888 -p 9876:9876 \
  -v ~/ultra_workdir:/opt/ultra_workdir quay.io/ultra.io/3rdparty-devtools:0.4.1-ubuntu24
# the same self-test inside the running container:
docker exec ultra bash -lc ultra-smoke
# compile: cdt-cpp inside the image (or the VS Code extension)
# test:    ultratest2 -t <spec>   (preinstalled — no `npm i -g` needed)
# dapp:    npm i @ultraos/wallet-sdk @wharfkit/antelope   (all public)
```

> ⚠️ **Port 8888 clash.** If a nodeos is already listening on the host's `:8888` (a native
> install, a leftover test chain, another container), `docker run -p 8888:8888` fails to bind
> and the container does not start. Stop it first (`pkill -x nodeos` on the host / `docker rm -f`
> the other container), or publish a different host port (`-p 18888:8888`) and point clients at
> that.

Ships (verified in both `0.4.1-ubuntu24` = Ubuntu 24.04.5 and `0.4.1-ubuntu22` = Ubuntu 22.04.5)
**nodeos/cleos v6.2.2-3.0.1 (Savanna; same 6.2.2 line as mainnet)**, **CDT 4.1.1** (`cdt-cpp`,
build `4.1.1-3.0.2`), **eosio.contracts 5.2.1**, **Node v22.11.0 / npm 10.9.0**,
**`@ultraos/ultratest2@1.0.6` preinstalled globally** (so `ultratest2 -t <spec>` just works;
the legacy v1 `@ultraio/ultratest@1.1.0` also sits at `/opt/ultratest`), the built **`eosio` system contracts** at
`/opt/eosio.contracts/build/contracts` (`eosio.system`, `eosio.token`, `eosio.nft.ft`,
`eosio.msig`, `eosio.oracle`, `eosio.group`), the Tip Jar
worked example at `/opt/templates/tipjar`, and `/usr/local/bin/ultra-smoke` (self-test). Build
inputs are recorded in `/opt/versions.json`. Validated 2026-09-25 — `ultra-smoke` green
(Tip Jar 6/6) on fresh pulls of both tags, and a clean-room agent built a new contract +
ultratest2 specs + Vue dapp from this KB and `0.4.1-ubuntu24` alone. Built + published by CI
(`ultra.docker` `external.yml`), so it refreshes reproducibly.

> **Tag scheme.** Releases are published as immutable, distro-qualified tags:
> **`<version>-ubuntu24`** (primary, Ubuntu 24.04) and **`<version>-ubuntu22`** (Ubuntu 22.04),
> plus **`<commit-sha>-ubuntu24`** / **`<commit-sha>-ubuntu22`**. **Pin a versioned tag**
> (`0.4.1-ubuntu24`). **`latest` moves only by an explicit promotion** — since 2026-09-25 it points at
> `0.4.1-ubuntu24`; the previous July-2026 build (nodeos v6.2.2-3.0.0, ultratest2 1.0.4) stays pullable as
> `912c1f58df838cfa29decac18f30ee8b8aee9955`. Whatever you pulled, `cat /opt/versions.json` says what's inside.
>
> **Upgrade note.** Before 2026-07-24 this image shipped **nodeos v5.0.2 (pre-Savanna)**,
> CDT 4.0.1, Node 19 and only ultratest **v1**. If you need that older image it remains at the
> digest tag **`618e324fc60de62b6e65d757340c45daadbbf868`** (and `0.2.0`). The `:0.3.1` tag pins
> the July-2026 Savanna build (nodeos v6.2.2-3.0.0, ultratest2 1.0.4); `0.4.1-*` supersedes it.

**Validated 2026-07-23** (full Tip Jar flow, docker-only, host toolchain untouched), then
**re-validated the same day, from published npm, with ZERO workarounds** after the tooling
fixes below shipped: compile with the image's `cdt-cpp` ✅; `npm i -g @ultraos/ultratest2`
runs the whole spec suite ✅ (6/6 green) against the image's nodeos + its
`/opt/eosio.contracts/build/contracts`. This requires **`@ultraos/ultratest2 ≥ 1.0.4`** and
**`@ultraos/ultra-signer-lib ≥ 1.7.5`** (both published 2026-07-23); a fresh `npm i -g`
resolves both automatically.

The only per-project setup is a **spec-directory `package.json`** — the runner refuses to start
without one (`package.json not found … Use either --create-test or --create-plugin`). Only its
`ultratestPlugins` block is actually required: on every run ultratest2 (verified 1.0.6) rewrites
`dependencies` so `@ultraos/ultratest` (an alias for its own `src`) and each listed native plugin
point at its install as **relative** paths, and it **runs `npm install` in the spec dir itself**
whenever it rewrote the file or `node_modules/` is missing. So **you never need to run
`npm install` in a spec dir** (doing it anyway is harmless). The fuller form below — what
`ultra-smoke` writes; `<GLOBAL_ROOT>` = `npm root -g` — works too:

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

Then just `ultratest2 -t <spec>`. The first run needs registry access and leaves a generated
`node_modules/` + `package-lock.json` (and a rewritten `package.json`) in the spec dir — gitignore
the first two.

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

**Still verify on testnet.** The image now runs a **Savanna chain on the same 6.2.2 line as
mainnet** (nodeos v6.2.2-3.0.1) with current system contracts, so a green local run is a far stronger
signal than before — but it is still a local single-node chain: re-verify on testnet before
mainnet (`08`).

`@ultraos/wallet-sdk` **≥ 0.3.2** (current 0.6.1) ships a bundled `dist/` + `exports` map, so it imports
cleanly in **plain Node / SSR / test runners** as well as bundlers. (On `0.3.1` and earlier a
plain-Node import fails with `ERR_UNSUPPORTED_DIR_IMPORT` — upgrade rather than work around it.)

## 4. Reading this KB without internal access

- Absolute paths under `/home/adam/...` describe the **internal reference machine** — on
  it, they are exact; elsewhere, treat them as "the private checkout of X" and use the
  public alternative above.
- References tagged `[internal]` point at private Ultra repos/docs; each is accompanied by
  enough inline context that no doc *depends* on following one.
- Everything else (chain IDs, endpoints, account model, RAM policy, wallet API,
  patterns, commands) is public information verified against public sources.
