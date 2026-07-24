# PLAN — Master Dev Image (`ultra-dev-all-in-one`)

**Status:** ✅ **SHIPPED** — Phase 1 quick wins EXECUTED 2026-07-23 (gaps 1,2,3,7 closed) and
the **public image refreshed 2026-07-24** (gaps 4,5 closed): `quay.io/ultra.io/3rdparty-devtools`
**`:latest` + `:0.3.0`** now ship **nodeos v6.2.2-3.0.0 (Savanna)**, Node 22, CDT 4.0.1,
`@ultraos/ultratest2@1.0.4` preinstalled, current contracts, `/opt/templates`, `ultra-smoke`
(green, Tip Jar 6/6 on a fresh pull). Pre-Savanna build preserved at tag `618e324f…`.
**wallet-sdk `exports` map SHIPPED 2026-07-24** (`@ultraos/wallet-sdk@0.3.2`, gap 6 closed).
Remaining: CI-pipeline reproducibility only (⚠️ `ultra.docker`'s `PAT_BECAUSE_APP_WASNT_WORKING`
secret is expired — see §7).
**Last Updated:** 2026-07-23
**Goal:** make the KB's promise true for the public: *an agent given this KB and Docker can
develop → build → test → launch a complete dapp on a local chain, and be walked to a
mainnet deploy, with ZERO private-repo access, ZERO compile-from-source, ZERO reliance on
a pre-provisioned machine.*

---

## 1. Why (validated gap summary)

The only public toolchain image today is `quay.io/ultra.io/3rdparty-devtools:latest`,
last published **2025-03-04** — one full protocol generation stale:

| Component | In the public image | Current (mainnet / internal) | Impact |
| --- | --- | --- | --- |
| nodeos / cleos / keosd | v5.0.2-3.0.0 (pre-Savanna) | v6.2.2-3.0.0 (Savanna live) | local chain ≠ mainnet behavior (finality, protocol features) |
| CDT | 4.0.1 | 4.1.1 | compile drift vs shipped contracts |
| System contracts | old build in `/opt/eosio.contracts/build/contracts` | current master | missing newer Ultra actions/tables |
| Test framework | ultratest v1 only | **ultratest2** (npm, public) | KB + all current internal specs are ultratest2 |
| Node.js | v19 (EOL) | v22 LTS line | ultratest2/tsx + Vite expect ≥20; npm-9 global installs error |

npm side is healthy: `@ultraos/{ultratest2,ultratest,wallet-sdk,ultra-signer-lib}` and
`@wharfkit/antelope` are all public.

## 2. The deliverable

One public image, published to **`quay.io/ultra.io/`** (the existing public org), e.g.
`quay.io/ultra.io/ultra-dev:latest` (+ semver tags matching the nodeos version, e.g.
`6.2.2`). Contents:

1. **Chain binaries** — nodeos, cleos, keosd at the **mainnet-matching version**
   (v6.2.2-3.0.0 today), Savanna-era genesis/config defaults.
2. **CDT 4.1.1** — `cdt-cpp` + cmake toolchain (`lib/cmake/cdt`) so both standalone
   `cdt-cpp` and CMake-based contract builds work.
3. **Built system contracts** at the canonical path `/opt/eosio.contracts/build/contracts`
   (current master build: bios/system/token/msig/nft.ft/oracle/eba/…), so
   `ultratest2 --contracts-dir-path=/opt/eosio.contracts/build/contracts` just works.
4. **Node 22 LTS** + **ultratest2 preinstalled globally** (`@ultraos/ultratest2`) **with
   its plugin set resolvable** (genesis/system/ultra-contracts/ultra-startup plugins) —
   whatever the validation showed about plugin packaging (§6) must be solved *inside* the
   image so a spec dir needs no network at all.
5. **Dapp deps warm cache (optional):** a global npm cache layer with
   `@ultraos/wallet-sdk`, `@wharfkit/antelope`, `vite`, `vue`, `vitest`, `@playwright/test`
   (+ `playwright install chromium --with-deps` for headless E2E in-container).
6. **Scaffold + smoke:** `/opt/templates/` holding the Tip Jar worked example (contract +
   spec + e2e_setup + dapp skeleton = KB doc `09` in file form) and a
   `/usr/local/bin/ultra-smoke` script that compiles the template, runs its spec, and
   exits 0 — CI for the image itself, and an agent's first sanity command.
7. Non-root user, `/opt/ultra_workdir` volume convention (docs parity), ports 8888/9876
   exposed.

Size target: ≤ ~2 GB (the current image is 633 MB; CDT + node + playwright add the bulk).
If Playwright bloats it, split `ultra-dev:slim` (1–4) and `ultra-dev:e2e` (1–6).

## 3. How it gets built (internal CI — the private repos stay private)

A small internal pipeline (GitLab CI in `ultraio/devops`, or GitHub Actions in the private
`ultraio` org) that:

1. Builds/uses release artifacts of Spring (Ultra fork), CDT, and `eosio.contracts` master
   (the repos stay private — only **binaries** ship, same licensing posture as the
   existing devtools image).
2. Assembles the Dockerfile (sketch):

   ```dockerfile
   FROM ubuntu:24.04
   COPY --from=spring-artifacts  /usr/local/bin/{nodeos,cleos,keosd} /usr/local/bin/
   COPY --from=cdt-artifacts     /usr/opt/cdt /usr/opt/cdt        # + PATH + cmake config
   COPY --from=contracts-build   /build/contracts /opt/eosio.contracts/build/contracts
   RUN <install node 22 via nodesource> \
    && npm i -g @ultraos/ultratest2 <plugin packages or bundled tarballs> \
    && npx playwright install chromium --with-deps            # e2e variant only
   COPY templates/ /opt/templates/
   COPY ultra-smoke /usr/local/bin/ultra-smoke
   RUN ultra-smoke   # image is self-tested at build time
   ```

3. Publishes to `quay.io/ultra.io/ultra-dev:{latest,<nodeos-semver>}`.
4. **Refresh triggers:** every mainnet nodeos upgrade, every CDT release, monthly cron
   (system-contracts refresh) — the staleness of the current image (17 months) is exactly
   the failure mode to design against. A `versions.json` inside the image records the
   build inputs for agents to assert against.

## 4. KB integration (once the image exists)

- `00-ACCESS_AND_SOURCES.md` §3 flips from "devtools image + caveats" to a single
  `docker run quay.io/ultra.io/ultra-dev` bootstrap; docs `02`/`04` gain the in-container
  command variants; `09` gains a "same flow, in Docker" appendix using `/opt/templates`.
- The KB's clean-room validation gets re-run **in the container** and the result recorded
  in the README (that is the definition of done for this plan).

## 5. Definition of done

A fresh machine with only Docker: an agent reading the KB completes the entire Tip Jar
flow (compile → spec suite green → keep-alive chain → dapp vitest/build → Playwright
green) **inside/against the container**, and doc `08` then routes a real mainnet deploy
(Pro Wallet + KYC + RAM + `cleos set contract` — cleos from the same image).

## 6. Empirical gap list (2026-07-23 docker-only validation)

Method: the complete Tip Jar flow was attempted with ONLY the public image + public npm
(host toolchain untouched). Original result: **compile ✅ (CDT 4.0.1), full ultratest2 spec
suite ✅ 6/6, ultratest v1 boot ✅ — but the ultratest2 path needed 4 undocumented
workarounds**. Those workarounds (gaps 1–3, 7) were **fixed the same day** and the flow
re-validated from published npm with **zero** workarounds (see the Status column). The
gaps, by severity:

| # | Gap | Severity | Fix | Status |
| --- | --- | --- | --- | --- |
| 1 | The 4 `ultratest-*-plugin` packages are not on npm; spec `package.json`s reference a private checkout; 2 bundled plugin dirs lack `package.json` | **BLOCKER** | Ship `package.json` for the bundled `genesis`+`system` plugins; document the canonical public spec `package.json` | ✅ **DONE** — ultratest2 **1.0.4** (PR #122) + `00` §3 recipe |
| 2 | `@ultraos/ultra-signer-lib@1.7.4` rejects `http://0.0.0.0:8888` → **every fresh ultratest2 install fails at genesis** (floating `^1.6.2` dep) | **BLOCKER — live regression** | Allowlist `0.0.0.0`, republish; raise ultratest2's floor | ✅ **DONE** — signer-lib **1.7.5** (PR #53), ultratest2 floor → `^1.7.5` (1.0.4) |
| 3 | Genesis plugin hardcodes `<contracts>/../../contracts/eosio.bios.1.8.3` (source layout); image ships only `build/contracts` | DEGRADED | Add a fallback in the plugin | ✅ **DONE** — bios fallback in ultratest2 1.0.4 |
| 4 | Image node 19 / npm 9: `npm i -g` errors (works by accident), engine warnings | DEGRADED | Node 22 LTS in the image; preinstall ultratest2 | ⏳ needs the §2 master image (npm-9 error is cosmetic today) |
| 5 | nodeos v5.0.2 pre-Savanna + old system contracts (no `ultra.bridge`, missing newer Ultra actions) | DEGRADED (latent) | The §2 image refresh; treat image-green as necessary-not-sufficient until then | ⏳ needs the §2 master image |
| 6 | `@ultraos/wallet-sdk@0.3.1` unloadable in plain Node — compiled `src/index.js` used directory/extensionless internal imports (`ERR_UNSUPPORTED_DIR_IMPORT` under Node ESM); bundlers resolved them | NIT (dapps) | Bundle to a single self-contained file + `exports`/`types` map | ✅ **DONE** — **0.3.2 published** (bundled `dist/` + exports map; identical API surface) |
| 7 | ultratest v1 interactive first-run prompt; ultratest2 docker-sock probe error in-container | NIT | Guard the probe | ✅ **DONE** — docker-sock probe silenced in ultratest2 1.0.4 |

### Quick wins that need NO new image — ✅ EXECUTED 2026-07-23

1. ✅ **`@ultraos/ultra-signer-lib@1.7.5`** published (PR #53) — allowlists `0.0.0.0` so
   fresh ultratest2 installs boot genesis. Fixed the live regression that broke internal
   machines too.
2. ✅ **`@ultraos/ultratest2@1.0.4`** published (PR #122) — signer floor `^1.7.5`,
   `package.json` for `genesis`+`system` plugins, bios-path fallback, docker-sock probe
   guard, canonical spec `package.json` documented. Also fixed the publish workflow's npm
   auth (`registry-url`, PR #123). **This turned the public path from "4 workarounds" into
   "npm i -g and go"** — re-validated from published npm, Tip Jar 6/6, zero workarounds.
3. ✅ **`@ultraos/wallet-sdk@0.3.2` published 2026-07-24 (gap 6 closed).** `0.3.1` shipped
   compiled JS whose `src/index.js` used directory/extensionless internal imports
   (`export * from './lib/interfaces'`), so a plain-Node/SSR/test-runner import died with
   `ERR_UNSUPPORTED_DIR_IMPORT`; bundlers resolved them, which is why dapps never saw it.
   **Fix:** bundle the internals into one self-contained ESM file, deps left external, plus an
   `exports`/`types` map — built from the **published 0.3.1 artifact** (NOT the 0.5.x working
   tree, which carries unreleased surface):

   ```bash
   esbuild src/index.js --bundle --format=esm --platform=neutral --packages=external \
     --outfile=dist/index.mjs
   ```
   `main`/`module` → `./dist/index.mjs`, `types`/`typings` → `./src/index.d.ts`, `exports` with
   `"."` + `"./package.json"`.

   **Validated before publishing:** identical export surface to 0.3.1
   (`UltraWalletSDK, PurchaseItemType, ResponseStatus, SdkErrorCode, SDK_ERROR_MESSAGE`);
   plain-Node import works on a fresh install of the published tag (0.3.1 still reproduces the
   error); **`ultra-tool-kit`** `vue-tsc && vite build` green; **`ultra-bridge-dapp`**
   `vite build` green + `vitest run` **1022/1022** green.

   **Scope note (corrects an earlier assumption):** the **browser extension and web wallet are
   not npm consumers** — web-app resolves `@ultraos/wallet-sdk` through a tsconfig path alias
   to `libs/wallet-sdk/src`, so an npm publish cannot affect them. Only `ultra-tool-kit`
   (`^0.3.1`) and `ultra-bridge-dapp` (`^0.3.0`) install from npm, neither uses deep imports
   (so the `exports` map is backward-compatible), and both already build/test green against
   0.3.2. No consumer manifest bump is required — their existing carets cover 0.3.2.

The §2 master image then removes the remaining drift (gaps 4–5) and makes the whole
toolchain one `docker pull`.

> **Publish-token note (2026-07-23):** the GitHub org `NPM_TOKEN` used by CI publishes
> `@ultraos/ultra-signer-lib` fine but **lacks write scope for `@ultraos/ultratest2`** (CI
> publish → `E404`), so 1.0.4 was published manually from a scoped account. **DevOps
> follow-up:** add `@ultraos/ultratest2` (read+write) to the org `NPM_TOKEN` so CI
> auto-publish works. (The `registry-url` gap in ultratest2's `publish.yml` is already
> fixed by PR #123.)

## 7. As-built — first manual master image (2026-07-23)

**Published:** `quay.io/ultra.io/eosio-docker-starter:ultra-dev-6.2.2`
(digest `sha256:f352b2f8…`, ~2.5 GB). **Validated:** `ultra-smoke` green — bundled Tip Jar
compiled with the image CDT, full spec suite **6/6** on a fresh Savanna genesis, zero
workarounds.

**Method (overlay, not from-scratch):** `FROM quay.io/ultra.io/3rdparty-devtools:latest`, then
swap in the mainnet-current pieces — this reuses the base's proven apt runtime-dep layer and
avoids shipping CDT's 2.9 GB build tree. Overlaid:

- **Chain binaries** → `nodeos/cleos/keosd v6.2.2-3.0.0` (Savanna) from the local Spring build
  (`/usr/local/bin`), runtime libs `libtinfo6 libgmp10 zlib1g libstdc++6`.
- **Node 19 → 22.11.0 LTS** — *wipe `/usr/local/lib/node_modules` first*, else npm 10 over the
  stale tree dies with `Class extends value undefined` on `npm i -g`.
- **System contracts** → `eosio.contracts` master build (25 contracts incl. the DeFi suite),
  plus `eosio.bios.1.8.3` copied from the source `contracts/` dir (the master `build/` omits it
  and genesis needs it → the ultratest2 1.0.4 bios fallback resolves it).
- **`@ultraos/ultratest2@1.0.4`** preinstalled globally (public npm).
- **`/opt/templates/tipjar`** (contract + spec) and **`/usr/local/bin/ultra-smoke`** self-test;
  **`/opt/versions.json`** records the build inputs.

**Deliberately kept CDT at 4.0.1** (from the base) — 4.1.1's build tree is 2.9 GB and 4.0.1
compiles the current contracts fine; a proper CDT-4.1.1 *install* tree (not the build tree) is
the follow-up.

**Remaining for a durable image:** (a) a CI-reproducible build — the `ultra.docker` pipeline
(`external.yml`) is the vehicle, but its version source (`blockchain-manager` main
`versions.json`) is pinned to nodeos **5.0.2**, so mainnet-currency there needs a
blockchain-manager bump (affects other consumers) + a dispatch; the node-22 + preinstall-ultratest2
Dockerfile step landed via `ultra.docker` PR #15. (b) an optional `:e2e` variant with Playwright
+ chromium for in-container dapp E2E (plan §2.5–2.6). (c) decide whether this becomes the KB's
default `docker pull` target and/or moves to a purpose-named repo/tag.
