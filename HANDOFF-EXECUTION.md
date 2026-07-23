# HANDOFF — Execute the public-tooling plan (cold pick-up)

**Created:** 2026-07-23 · **Status:** READY TO EXECUTE (Phases 1 & 3 fully; Phase 2 has 2 operator gates)
**Mission:** make Ultra dapp development fully self-service on public tooling — an agent
with this KB + Docker completes contract → tests → dapp → E2E with **zero private-repo
access, zero source compiles, zero workarounds**, then is routed to a mainnet deploy.
**Context docs (read first):** this repo's `README.md` → `00-ACCESS_AND_SOURCES.md` §3
(the 4 validated workarounds = the bug list) → `PLAN-MASTER_DEV_IMAGE.md` (§6 gap table).

## 0. Current state (done — don't redo)

- KB written, clean-room-validated (Tip Jar, all gates green), split to this standalone
  scrubbed repo (`gitlab.com/ultraio/devops/ultra-agent-kb`, private).
- **Docker-only validation done 2026-07-23:** full Tip Jar flow green using ONLY
  `quay.io/ultra.io/3rdparty-devtools:latest` + public npm — with 4 workarounds (in `00`
  §3). Validation scratch logs were session-temp (gone); the method is reproducible: §4.
- Tip Jar artifacts: contract+specs `/home/adam/spring/eosio.contracts-tipjar` (branch
  `example/tipjar`, local-only), dapp `/home/adam/ultra.repos/ultra-tipjar-dapp`.

## 1. Phase 1 — quick wins (no new image; do in this order)

### P1.1 Fix the signer-lib regression 🔴 (breaks every fresh ultratest2 install TODAY)

- Repo: `/home/adam/ultra.repos/ultra-signer-lib` (github.com/ultraio/ultra-signer-lib,
  private). Published `@ultraos/ultra-signer-lib@1.7.4` added `validateEndpoint` —
  **`src/api/index.ts:8-18`**: rejects non-HTTPS unless hostname ∈
  `['localhost','127.0.0.1','[::1]']`. ultratest2 announces `http://0.0.0.0:8888` → every
  genesis boot throws. (1.7.3 has no check; confirmed by tarball diff.)
- **Fix:** add `'0.0.0.0'` to the allowlist (0.0.0.0 as a client target = this-host on
  Linux; dev-only semantics, no security regression — HTTPS still required for real
  hosts). Also rebuild `lib/` (the check is duplicated in the shipped JS).
- Flow: branch → PR → merge → publish 1.7.5 via the repo's `.github/workflows/publish.yml`
  (GitHub Actions; npm token is org-managed — if publish fails on auth, that's the
  `@ultraos` org npm token, see internal `npm-token/README.md` in ultraOS-doc).
- **Verify:** on any machine WITHOUT the workaround pin:
  `npm i -g @ultraos/ultratest2 && ultratest2 --contracts-dir-path=<contracts> -t <any spec>`
  boots genesis without `Non-HTTPS endpoint rejected`.

### P1.2 Republish `@ultraos/ultratest2` (kills workarounds 2–4)

- Repo: `/home/adam/ultra.repos/ultratest2` (github.com/ultraio/ultratest2, private;
  npm publish via `.github/workflows/publish.yml`; current 1.0.3, deps
  `@ultraos/ultra-signer-lib: ^1.6.2`).
- Plugin reality: the 4 plugins live in-tree at `src/plugins/native/{genesis,system,
  ultraContracts,ultraStartup}` and are ALSO synced to standalone **GitHub repos**
  (`ultraio/ultratest-ultra-startup-plugin` etc., via `release-*-plugin.yml`) — they are
  **not on npm**, and `genesis`/`system` dirs lack a `package.json` in the tarball.
- Changes (one PR):
  1. Dep floor: `@ultraos/ultra-signer-lib: "^1.7.5"` (after P1.1; or pin exact).
  2. Add proper `package.json` to `src/plugins/native/genesis` + `system` (name/version/
     main), matching the stubs the other two carry — so `file:` deps work from the
     published tarball.
  3. Genesis plugin bios fallback: where it scandirs
     `<contracts-dir>/../../contracts/eosio.bios.1.8.3`, fall back to
     `<contracts-dir>/eosio.bios.1.8.3` (docker image layout).
  4. Guard the `/var/run/docker.sock` purge probe (skip silently when absent).
  5. Docs: add to the README the canonical PUBLIC spec-dir `package.json` (deps via
     `file:` into the installed package — the shape that worked is in KB `00` §3 item 3).
- **Verify:** repeat §4 in a fresh container with ZERO manual workarounds → 6/6 green.
- Optional stretch: also publish the 4 plugins as real npm packages (their GitHub sync
  repos already exist; adds discoverability, not required once the tarball is fixed).

### P1.3 `@ultraos/wallet-sdk` exports map (plain-Node importability)

- Source: GitLab `ultraio/web-app` monorepo, `libs/wallet-sdk` (source tree is at 0.5.0;
  published npm is 0.3.1 shipping raw `src/` — no `exports` map, directory imports →
  unloadable outside bundlers).
- Fix: publish transpiled `dist/` + `exports` map (+ types). ⚠️ Follow the wallet-system
  release protocol — internal `web-browser-extension/WALLET_SYSTEM_AGENT_CONTEXT.md` §8.4
  (coordinated multi-MR; consumers pin `^0.3.1`) — and don't ship unreleased 0.5.x surface
  by accident; a `0.3.2` patch-publish from the release branch is the safe shape.
- Lowest priority: bundler-based dapps (the KB's recommended stack) are unaffected.

## 2. Phase 2 — the master image (`PLAN-MASTER_DEV_IMAGE.md` §2–3 is the spec)

Prereqs from Phase 1: P1.1+P1.2 shipped (the image preinstalls the fixed ultratest2).

**Operator gates — ASK before proceeding:**
1. **quay.io push credentials** for `quay.io/ultra.io` (who owns the org? the existing
   devtools image was last pushed 2025-03-04 — find that pipeline first; likely an
   internal repo named like `3rdparty-devtools`/`docker-*` in GitHub `ultraio`).
2. **CI placement decision** (GitHub Actions in private `ultraio` vs GitLab
   `ultraio/devops`) + approval to publish mainnet-version binaries publicly (same
   licensing posture as the existing image, but confirm).

Build steps once gated: follow the plan's §2 contents list + §3 Dockerfile sketch;
artifact sources on this machine for a first manual build: Spring build
`/home/adam/spring/spring/build/bin/`, CDT `/home/adam/spring/eosio.cdt/build`, contracts
`./build.sh -c ../eosio.cdt/build` output from a CURRENT `eosio.contracts` master
checkout (use a fresh worktree; don't touch `eosio.contracts` master WIP or `-defi`).
Include `/opt/templates/` = the Tip Jar files (copy from the §0 artifact paths) + an
`ultra-smoke` script. Manual first build + push is acceptable to unblock; the CI pipeline
(+ refresh triggers per plan §3.4) is the durable deliverable.

## 3. Phase 3 — KB updates + final validation (definition of done)

1. Update `00` §3: delete the 4 workarounds (keep as a "pre-2026-08 versions" footnote);
   point at the new image/bootstrap. Update `02`/`04`/`09` per plan §4; update
   `PLAN-MASTER_DEV_IMAGE.md` status → EXECUTED with a dated as-built section.
2. **Re-run the clean-room validation**: a fresh agent, KB-only, Docker-only (host
   toolchain forbidden), full Tip Jar flow — must pass with ZERO workarounds. Record the
   result in the KB README. That closes the mission.
3. Update ultraOS-doc `DOCUMENTATION_INDEX.md` KB entry + the auto-memory
   (`project_agent_knowledge_base_ultra_dev.md`) with the new state.

## 4. Reproducible validation recipe (used for the 2026-07-23 run — reuse for verifies)

```bash
docker run -dit --name kbval --entrypoint /bin/bash -p 18899:8888 \
  -v <scratch>:/work quay.io/ultra.io/3rdparty-devtools:latest    # (or the new image)
# copy in: eosio.contracts-tipjar/contracts/tipjar + ultratests/tipjar (+ repoint the
#   spec helper's publishContract path at the in-container artifact dir)
# compile:  cdt-cpp -abigen -I include -o tipjar.wasm src/tipjar.cpp
# test:     npm i -g @ultraos/ultratest2
#           ultratest2 --contracts-dir-path=/opt/eosio.contracts/build/contracts \
#             -t /work/tipjar/tipjar.spec.ts > /work/spec.log 2>&1
# expect:   6/6 cases green, RUN_EXIT=0; then docker rm -f kbval
```

Old-image quirks (disappear with the new image): default entrypoint mangles args → always
`--entrypoint /bin/bash`; ultratest v1 first-run prompt → pipe a newline; host :8899 may
collide with a host keosd → use 18899.

## 5. Working rules

- Multi-agent machine: own worktree/branch per repo; never commit to a shared checkout's
  primary branch; **never touch** `/home/adam/spring/eosio.contracts` (master WIP),
  `eosio.contracts-defi`, or the Tip Jar artifacts (reference material).
- GitHub `ultraio` repos: PR flow; GitLab: MR flow (token = ultra-devops account — a 403
  means scope, not expiry). Never read/print tokens.
- Host toolchain must stay untouched during validations — that's the point of the test.
- Sequencing: P1.1 → P1.2 → (P1.3 anytime) → Phase 2 → Phase 3. P1.1+P1.2 alone already
  turn the public path into "npm i -g and go" on the OLD image — ship them even if
  Phase 2 stalls on its gates.
