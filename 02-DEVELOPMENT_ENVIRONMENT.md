# 02 — Development Environment

**Last Updated:** 2026-07-24
**Read this to:** know exactly what toolchain exists (and where it is on the internal
reference machine), and how to stand it up elsewhere. **No private access?** Use the
public bootstrap in `00` §3 (quay.io devtools image + npm packages) instead of §1.

---

## 1. The toolchain, verified on this machine

| Tool | Version | Location | Purpose |
| --- | --- | --- | --- |
| **CDT** (`cdt-cpp`) | 4.1.1 | `/usr/local/bin/cdt-cpp`; build tree `/home/adam/spring/eosio.cdt/build` | compile C++ → wasm/abi |
| **nodeos / cleos** | v6.2.2-3.0.0 (Ultra Spring fork) | `/usr/local/bin/{nodeos,cleos}` (byte-identical to `/home/adam/spring/spring/build/bin/nodeos`) | local chain + CLI |
| **ultratest2** | `@ultraos/ultratest2` (public npm `latest` ≥ **1.0.4**; internal machine runs a source symlink) | `~/.nvm/versions/node/v22.21.0/bin/ultratest2` → symlink to `/home/adam/ultra.repos/ultratest2` | contract test framework (TS, runs via `tsx`, no build step). Public bootstrap: `npm i -g @ultraos/ultratest2` — see `00` §3 |
| **Node.js** | v22.21.0 (nvm) | `~/.nvm` | ultratest2, dapps |
| Spring source | `/home/adam/spring/spring` (also `/home/adam/spring/eosio`) | | protocol reference |
| eosio.contracts | `/home/adam/spring/eosio.contracts` (branch `master`) | | system contracts + `build.sh` |
| **eosio.contracts-defi** | `/home/adam/spring/eosio.contracts-defi` — **git worktree of the same repo**, branch `feature/ultra-dex-amm` | | the DeFi suite: the best real contract+test exemplars |

⚠️ **Worktree trap** `[internal]`**:** older docs (ultraOS-doc `ultra-defi/AGENT_CONTEXT.md`,
`DEMO_AND_LAUNCH_GUIDE.md`) cite `/home/adam/spring/eosio.contracts/...` paths for the DeFi
contracts. Since 2026-06-16 those live in the **`eosio.contracts-defi` worktree** —
substitute the path in every copied command. The main checkout is on `master` and carries
unrelated WIP; **don't build your work there** — make your own worktree (`03` §2).

## 2. Sanity checks (run these before starting work)

```bash
cdt-cpp --version                 # cdt-cpp version 4.1.1
nodeos --version                  # v6.2.2-3.0.0
ultratest2 --version              # banner (NEVER run `ultratest2 --help` — it hangs)
ls /home/adam/spring/eosio.contracts-defi/build/contracts/ultra.dex/   # prebuilt exemplar wasm/abi
```

A leftover `nodeos` from a previous session blocks new test chains:
`pkill -x nodeos` first if port `:8888` is busy (ultratest2 also pkills on start).

## 3. Standing it up from scratch (another machine)

1. **CDT 4.1.1** — build `eosio.cdt` (cmake) or install Ultra's package; the contract
   build needs `<cdt>/lib/cmake/cdt/cdt-config.cmake` to exist.
2. **Spring (Ultra fork)** — build `github.com/ultraio/spring` (private; public path =
   the devtools image, `00` §3); install `nodeos`/`cleos`
   into `/usr/local/bin` (ultratest2 execs bare `nodeos` from `$PATH`).
3. **Node 20+ via nvm**, then install ultratest2 globally (`npm i -g @ultraos/ultratest2`
   or clone `ultra.repos/ultratest2` and link). Specs need **no per-directory
   `npm install`** — test dirs reference the ultratest2 checkout via relative paths.
4. **eosio.contracts** — clone `github.com/ultraio/eosio.contracts`; `./build.sh -c
   <cdt-build-dir>` once to produce `build/contracts/*` (system contracts are needed by the
   test chain bootstrap).
5. The official *public* developer path is different (VS Code extension `ultraio.ultra-cpp`
   + Docker images + cleos — see `docs-blockchain/docs/tutorials/smart-contracts/`). Both
   work; this KB teaches the native `build.sh` + ultratest2 path used by the shipped
   DeFi suite because it is scriptable and agent-friendly.

## 4. Dapp-side toolchain

- Vite + Vue 3 + TypeScript (exemplars: `/home/adam/ultra.repos/ultra-dex-dapp`,
  `ultra-lend-dapp`, `ultra-farm-dapp`; React variant: `ultra-bridge-dapp`).
- `@ultraos/wallet-sdk` (published **0.3.2** — pin `^0.3.2`, see `05` §1) + `@wharfkit/antelope` (^1.0.13).
- Playwright for E2E; vitest for unit tests.
- npm registry access required (a corporate VPN can block npm).
- For real-extension QA you need Chrome + the extension built from
  `/home/adam/ultra.repos/web-app` (`06` §7).
