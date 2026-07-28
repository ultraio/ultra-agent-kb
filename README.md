# Ultra Developer Knowledge Base — for AI Agents

**Last Updated:** 2026-07-24
**Purpose:** give an AI agent (or a new developer) everything needed to go from a one-line
prompt — *"build me a dapp that does X on Ultra"* — to a working smart contract, a tested
web dapp integrated with the Ultra Wallet, and a deployment path to testnet/mainnet, without
re-discovering the platform from scratch.

**How to use this KB:** read `01` once (platform mental model), then follow the task router
below. **If you lack Ultra-internal access, read `00` first** — it maps every referenced
repo/tool to its public equivalent (npm packages, quay.io images, developers.ultra.io);
references into private material are tagged `[internal]` throughout. Every doc is self-contained, states exact commands, and cites source files in the
sibling repos on this machine. When a doc conflicts with code, the code wins — fix the doc.

---

## The expected development flow

1. **Feed the agent this KB**: *"Read `README.md` in ultra-agent-kb, then: build me a dapp
   that does X."* The router below tells it what to read for each phase.
2. **Bootstrap tooling** — internal machine: `02` §1 (native toolchain). Public/fresh
   machine: `00` §3 (docker image + npm; no private repos, no compiling from source).
   **The only host prerequisite for the public path is a working Docker daemon — verify
   `docker info` FIRST. If it fails, Tier 2 (contract compile + local chain) cannot run on
   this machine, and that is an environment limitation, not a KB gap.**
3. **Develop locally**: contract (`03`) → spec suite green on a real local chain (`04`) →
   dapp + wallet integration (`05`/`06`) → Playwright E2E against a keep-alive seeded
   chain (`04` §6, `05` §6). The Tip Jar (`09`) is the full worked template.
4. **Ship**: testnet first, then mainnet (`08` — permissionless deploy: account + UOS + RAM; runbook,
   governance handoff), dapp hosting (`08` §6). Verify per `08` §7.

Everything a dapp needs at runtime (wallet SDK, read client) is public npm; everything the
dev loop needs is in `00`'s matrix.

## Task router — read what you need

| Your task | Read (in order) |
| --- | --- |
| Understand how Ultra works (chain, accounts, resources, Uniqs) | `01` |
| Set up / verify the dev toolchain | `02` |
| Write a smart contract | `01` → `03` → `04` |
| Test a contract (local chain) | `04` |
| Build a dapp frontend | `05` → `06` |
| Integrate the Ultra Wallet (extension / web wallet) | `06` |
| Read chain data (RPC, GraphQL, indexers) | `07` |
| Deploy to testnet / mainnet (contract + dapp) | `08` |
| See one complete end-to-end example | `09` |
| Avoid known traps | `10` (checklist; each doc has its own gotchas too) |
| Integrate a core contract (`eosio.token`, Uniqs/`eosio.nft.ft`, msig, oracle) | `11` |
| Know what's public vs private (repos, images, packages) | `00` |

## Contents

- **`00-ACCESS_AND_SOURCES.md`** — the access matrix: which referenced repos are private,
  the public distribution channels (npm `@ultraos/*`, `quay.io/ultra.io/*` images,
  developers.ultra.io), and the no-private-access bootstrap path.
- **`01-ULTRA_CHAIN_FUNDAMENTALS.md`** — what Ultra is (Antelope/Spring L1), chain IDs +
  endpoints, account model (EBA + named accounts), keys/permissions, the Ultra resource
  model (RAM/CPU/NET — how it differs from vanilla EOSIO), UOS, Uniq NFT standard
  (eosio.nft.ft), system-contract inventory, the oracle, governance, and every
  Ultra-vs-vanilla-EOSIO difference that changes how you build.
- **`02-DEVELOPMENT_ENVIRONMENT.md`** — the toolchain (CDT, Spring/nodeos, ultratest2,
  node/npm), where it is installed on this machine, versions, and how to stand it up from
  scratch.
- **`03-SMART_CONTRACT_DEVELOPMENT.md`** — contract project layout, a minimal working
  contract, actions/tables/ABI, the transfer+memo-dispatch pattern, inline actions,
  RAM-payer rules, security patterns (effects-before-interactions, closing assertions,
  spoofed-token rejection), and the hard-won platform gotchas.
- **`04-CONTRACT_TESTING_ULTRATEST2.md`** — ultratest2 from zero: spec anatomy, chain
  bootstrap, running one spec, `--keep-alive`, e2e seeding, key management, oracle
  seeding, assertion helpers.
- **`05-DAPP_DEVELOPMENT.md`** — the proven dapp stack (Vue 3 + Vite + TS), project
  structure, reading chain state, the math-mirror pattern, unit + E2E testing.
- **`06-WALLET_INTEGRATION.md`** — the Ultra Wallet ecosystem (extension, web wallet),
  `window.ultra` API, `@ultraos/wallet-sdk`, connect/sign/broadcast flows, events,
  network sync, local-dev setup, Playwright wallet mocks.
- **`07-CHAIN_INTERACTION_AND_DATA.md`** — live RPC endpoints (mainnet + testnet), cleos
  recipes, the public GraphQL API, dfuse/firehose, block explorers.
- **`08-TESTNET_AND_MAINNET_DEPLOYMENT.md`** — accounts + RAM, deploying a contract
  (testnet, then mainnet — permissionless deployment), msig, code-lock /
  immutability, and the standard dapp-hosting pattern (Cloudflare Pages).
- **`09-WORKED_EXAMPLE_TIP_JAR.md`** — a complete, actually-built example: contract +
  ultratest2 spec + Vue dapp with wallet signing, every command, every gotcha hit.
- **`10-PITFALLS_CHECKLIST.md`** — the condensed "before you ship" checklist.
- **`11-CORE_CONTRACT_INTERFACES.md`** — integrating Ultra's system contracts whose **source is
  private but whose interface is public**: how to pull any contract's ABI (image / `cleos get
  abi` / `/v1/chain/get_abi`), how to read it, and the verified action+table shapes and
  integration patterns for `eosio.token` (incl. the `on_notify` transfer pattern) and
  `eosio.nft.ft` (Uniq factory→token model).

## Prompt patterns that work

- *"Read `ultra-agent-kb/README.md`, then build and test a contract that …"*
- *"Read `ultra-agent-kb/README.md`, then build a dapp that … using the worked
  example (`09`) as the template."*

## Scope & provenance

This repo is deliberately **standalone**: it contains no credentials and no sensitive
operational data, so it can be shared more widely than Ultra's internal docs. Deep-dive
references into Ultra-internal material (the private `ultraOS-doc` repo, private source
repos) are tagged `[internal]` and are never load-bearing — every doc carries the facts it
needs inline, and `00` gives public alternatives. Absolute `/home/...` paths describe the
internal reference dev machine the KB was validated on; public readers substitute the
`00` §3 bootstrap. Content was synthesized from developers.ultra.io, the Ultra source
repos, and the shipped wallet/dapp codebases, then **clean-room-validated** (2026-07-23) by
an agent that built the complete Tip Jar stack (`09`) from this KB alone.

**Clean-room re-validated on the public path (2026-07-24).** A fresh agent, given only this
KB + `quay.io/ultra.io/3rdparty-devtools:latest` + public npm — with every host tool and every
other path on the machine forbidden — designed and shipped a **new** contract (a `pollbooth`
voting contract, deliberately not the Tip Jar: secondary index, numeric scope, `bool` column)
and its dapp: `cdt-cpp` build clean first try, **ultratest2 spec suite 7/7**, dapp unit tests
**16/16**, `vue-tsc + vite build` clean, and **5/5 live-chain integration** including a real
signed action and a surfaced contract assert. **No private repo, no host binary, no
workaround.** It reached doc `08`'s mainnet runbook and verified its commands against live
testnet. The 13 documentation defects it found are fixed in this revision (the notable ones:
`bool` fields deserialize as `1`/`0`; `09`'s build commands were internal-only; `08` omitted
the `cleos wallet` prerequisite).

**Public path proven (2026-07-23):** the docker-only bootstrap (`00` §3) was re-validated
from **published npm** with **zero workarounds** — a fresh `quay.io/ultra.io/3rdparty-devtools`
container, `npm i -g @ultraos/ultratest2` (`1.0.4`, which pulls `ultra-signer-lib 1.7.5`),
Tip Jar spec suite **6/6 green**. The four workarounds the earlier validation needed were
fixed and shipped that day (`@ultraos/ultra-signer-lib@1.7.5`, `@ultraos/ultratest2@1.0.4`);
see `PLAN-MASTER_DEV_IMAGE.md` §6. Both follow-ups then shipped too: the **refreshed
Savanna image** (2026-07-24, CI-built) and **`@ultraos/wallet-sdk@0.3.2`**.
