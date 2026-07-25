# CLOSEOUT — the public-tooling plan (EXECUTED)

**Created:** 2026-07-23 · **Executed:** 2026-07-23 → 2026-07-24 · **Status:** ✅ **COMPLETE**
**Mission (achieved):** make Ultra dapp development fully self-service on public tooling — an
agent with this KB + Docker completes contract → tests → dapp → E2E with **zero private-repo
access, zero source compiles, zero workarounds**, then is routed to a mainnet deploy.

> This file was the cold pick-up execution plan. Everything in it shipped; it is kept as the
> record of what was done and what remains for others. Current state lives in
> `PLAN-MASTER_DEV_IMAGE.md` (§6 gap table, §7 as-built, §8 clean-room re-validation).

---

## 1. What shipped

### Published packages (npm, all public)

| Package | Version | What it fixed |
| --- | --- | --- |
| `@ultraos/ultra-signer-lib` | **1.7.5** | `validateEndpoint` rejected ultratest2's `http://0.0.0.0:8888` → **every fresh install died at genesis**, internal machines included. Added `0.0.0.0` to the local-dev allowlist (HTTPS still mandatory for real hosts) + offline regression tests. GitHub PR #53. |
| `@ultraos/ultratest2` | **1.0.4** | Signer floor `^1.7.5`; `package.json` for the bundled `genesis`+`system` native plugins; genesis bios-path fallback for flat contract layouts; docker-sock purge-probe guard; canonical spec-dir `package.json` documented. PR #122 (+ #123 fixing `publish.yml`'s missing `registry-url`). |
| `@ultraos/wallet-sdk` | **0.3.2** | `0.3.1` shipped directory/extensionless internal imports → `ERR_UNSUPPORTED_DIR_IMPORT` in plain Node/vitest/SSR. Bundled to a single self-contained ESM file + `exports`/`types` map, built from the **published 0.3.1 artifact** (not the 0.5.x tree). |

### The public dev image

`quay.io/ultra.io/3rdparty-devtools` **`:latest` + `:0.3.1`** — refreshed 2026-07-24 from a
17-month-stale pre-Savanna build to:

- **nodeos/cleos/keosd v6.2.2-3.0.0 (Savanna, mainnet-matching)**, **CDT 4.1.1**,
  **eosio.contracts 5.1.0**, **Node 22**
- **`@ultraos/ultratest2@1.0.4` preinstalled** (no `npm i -g` needed)
- `/opt/templates/tipjar` (worked example) + **`/usr/local/bin/ultra-smoke`** (self-test) +
  `/opt/versions.json` (build inputs)
- Pre-Savanna build preserved at tag `618e324fc60de62b6e65d757340c45daadbbf868` / `0.2.0`

**Built and published by CI** — `ultraio/ultra.docker` → `.github/workflows/external.yml`
(`workflow_dispatch`) → buildah → quay. Supporting merges: **ultra.docker #15** (Node 22 +
preinstall ultratest2), **#16** (`download.js`: contents-API fetch + fail-loudly), **#17**
(templates + `ultra-smoke` + `versions.json`); **blockchain-manager #61** (`versions.json` →
eosio `6.2.2-3.0.0`, cdt `4.1.1-3.0.0`, contracts `5.1.0`).

### Two broken secrets found and fixed

The image had not rebuilt since **2025-03-04** — not because nobody ran the pipeline, but
because it was **silently broken in two places**:

1. `ultra.docker`'s **`PAT_BECAUSE_APP_WASNT_WORKING`** was expired, and `download.js`
   swallowed the failure into `TypeError: Cannot convert undefined or null to object`
   (fixed in #16 — it now throws with repo/ref/status).
2. `ultra.docker`'s **`QUAY_USERNAME`/`QUAY_PASSWORD`** were invalid → `Push To quay.io`
   failed with `Invalid Username or Password`.

Both rotated; a full `external.yml` dispatch is now green end-to-end.

## 2. Validation record

| Date | What | Result |
| --- | --- | --- |
| 2026-07-23 | Tip Jar flow, docker-only, host toolchain untouched | 6/6 — but needed **4 workarounds** |
| 2026-07-23 | Re-run after signer-lib 1.7.5 + ultratest2 1.0.4 published | **6/6, ZERO workarounds** |
| 2026-07-24 | `ultra-smoke` on the CI-published `:latest` (fresh pull) | **PASS**, Tip Jar 6/6 |
| 2026-07-24 | **Clean-room: new contract + dapp, KB + image + public npm only** | **PASS** — spec 7/7, dapp unit 16/16, build clean, live-chain integration 5/5 · **13 doc defects found → all fixed** |

Details of the last one: `PLAN-MASTER_DEV_IMAGE.md` §8.

## 3. What remains (for whoever picks this up)

Nothing blocks the mission. Open, in rough priority order:

1. **Swap the CI quay credential for a robot token.** `external.yml` currently pushes as a
   personal account (`adam_47`); a quay robot account is the durable choice.
2. **Automate image refresh.** Today it is a manual `workflow_dispatch`. Wire the refresh
   triggers from §3.4: every mainnet nodeos upgrade, every CDT release, plus a monthly cron.
   Note `blockchain-manager` `main`'s `versions.json` is the version source — bumping it is
   what makes a rebuild pick up new binaries.
3. **Optional `:e2e` image variant** with Playwright + chromium for in-container browser E2E
   (plan §2.5–2.6). The clean-room run could not do real-extension browser E2E without a GUI.
4. **`08` non-self-service residue** — mainnet KYC/KYB has no documented SLA, there is no
   "how to acquire UOS on mainnet", and §5 is a runbook shape rather than a worked mainnet
   transcript. Fixing these needs business input, not engineering.
5. **Read-only actions** (`03` §5.12) are recommended by the docs but have no verified
   invocation recipe from ultratest2/WharfKit — either prove one or stop recommending them.

## 4. Reproducible validation recipe (reuse for future verifies)

```bash
docker run -dit --name kbval --entrypoint /bin/bash -v <scratch>:/work \
  quay.io/ultra.io/3rdparty-devtools:latest
docker exec kbval bash -lc ultra-smoke      # expect: Tip Jar 6/6, "ultra-smoke: PASS"
# then, for a real clean-room test, build something NEW (not the bundled template):
#   compile:  cdt-cpp -abigen -I include -o /work/build/x/x.wasm src/x.cpp
#   spec dir: package.json per 00 §3 (file: paths into $(npm root -g))
#   test:     ultratest2 --contracts-dir-path=/opt/eosio.contracts/build/contracts -t /work/x/x.spec.ts
docker rm -f kbval
```

## 5. Working rules that applied

- Multi-agent machine: own worktree/branch per repo; never commit to a shared checkout's
  primary branch. **Never touch** `/home/adam/spring/eosio.contracts` (master WIP),
  `eosio.contracts-defi`, or the Tip Jar artifacts.
- GitHub `ultraio` repos: PR flow. GitLab: MR flow (token = ultra-devops; a 403 means scope,
  not expiry). **Never read or print a credential** — pipe to stdin only.
- Host toolchain stays untouched during validations — that is the point of the test.
