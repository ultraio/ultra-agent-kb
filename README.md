# Ultra Developer Knowledge Base — for AI Agents

**Last Updated:** 2026-09-25
**Purpose:** give an AI agent (or a new developer) everything needed to go from a one-line
prompt — *"build me a dapp that does X on Ultra"* — to a working smart contract, a tested
web dapp integrated with the Ultra Wallet, and a deployment path to testnet/mainnet, without
re-discovering the platform from scratch.

**How to use this KB:** read `01` once (platform mental model), then follow the task router
below. **If you lack Ultra-internal access, read `00` first** — it maps every referenced
repo/tool to its public equivalent (npm packages, quay.io images, developers.ultra.io);
references into private material are tagged `[internal]` throughout. Every doc is self-contained, states exact commands, and cites source files by
repo-relative path in Ultra's source repos. When a doc conflicts with code, the code wins — fix the doc.

---

## The expected development flow

1. **Feed the agent this KB**: *"Read `README.md` in ultra-agent-kb, then: build me a dapp
   that does X."* The router below tells it what to read for each phase.
2. **Bootstrap tooling** — with private repo access: `02` §1 (native toolchain). Public/fresh
   machine: `00` §3 (docker image + npm; no private repos, no compiling from source).
   **The only host prerequisite for the public path is a working Docker daemon — verify
   `docker info` FIRST. If it fails, Tier 2 (contract compile + local chain) cannot run on
   this machine, and that is an environment limitation, not a KB gap.**
3. **Develop locally**: contract (`03`) → spec suite green on a real local chain (`04`) →
   dapp + dual-provider wallet integration (`05`/`06`) → provider-branch tests + Playwright
   E2E against a keep-alive seeded chain (`04` §6, `05` §6). The Tip Jar (`09`) is the
   worked template; follow `06`'s real-wallet acceptance gate before claiming production support.
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
| Integrate the Ultra Wallet SDK (both extension + Web Wallet) | `06` (capability matrix and acceptance gate are mandatory) |
| Read chain data (RPC, GraphQL, indexers) | `07` |
| Deploy to testnet / mainnet (contract + dapp) | `08` |
| See one complete end-to-end example | `09` |
| Avoid known traps | `10` (checklist; each doc has its own gotchas too) |
| Integrate a core contract (`eosio.token`, Uniqs/`eosio.nft.ft`, msig, oracle) | `11` |
| Secure a contract (auth, fake-deposit, inline-ordering, attack classes) | `03` → `12` |
| Upgrade a live contract / add table fields / handle a breaking change + migrate data | `03` → `13` |
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
  node/npm), where it is installed in Ultra's internal setup, versions, and how to stand it up from
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
- **`06-WALLET_INTEGRATION.md`** — the canonical dual-provider `@ultraos/wallet-sdk`
  integration: extension/Web capability matrix, provider-aware wrapper, connect/sign/error
  flows, lifecycle split, environment availability, tests, and the production acceptance gate.
- **`07-CHAIN_INTERACTION_AND_DATA.md`** — live RPC endpoints (mainnet + testnet), cleos
  recipes, the public GraphQL API, dfuse/firehose, block explorers.
- **`08-TESTNET_AND_MAINNET_DEPLOYMENT.md`** — accounts + RAM, deploying a contract
  (testnet, then mainnet — permissionless deployment), msig, code-lock /
  immutability, and the standard dapp-hosting pattern (Cloudflare Pages).
- **`09-WORKED_EXAMPLE_TIP_JAR.md`** — a complete, actually-built example: contract +
  ultratest2 spec + Vue dapp with SDK-based extension/Web provider selection and wallet signing.
- **`10-PITFALLS_CHECKLIST.md`** — the condensed "before you ship" checklist.
- **`11-CORE_CONTRACT_INTERFACES.md`** — integrating Ultra's system contracts whose **source is
  private but whose interface is public**: how to pull any contract's ABI (image / `cleos get
  abi` / `/v1/chain/get_abi`), how to read it, and the verified action+table shapes and
  integration patterns for `eosio.token` (incl. the `on_notify` transfer pattern) and
  `eosio.nft.ft` (Uniq factory→token model).
- **`12-SMART_CONTRACT_SECURITY.md`** — the consolidated security ruleset: authorization
  correctness (gate the party who bears the cost), the fake-deposit three-part guard, the
  inline/notification execution-ordering hazard (you can't see an inline's state change
  mid-action), and a named catalogue of the historical Antelope/EOSIO attack classes with each
  one's status on Ultra. Grounded in the live system contracts + Spring protocol source.
- **`13-CONTRACT_UPGRADES_AND_BREAKING_CHANGES.md`** — how to evolve a **deployed** contract's
  storage without corrupting on-chain data: the append-only table-layout rule (never
  rename/retype/re-mean/remove a field), adding fields safely via **binary extension**, and —
  when a change IS breaking — the versioned-table + data-migration playbook (`_v0`/`.a` →
  `_v1`/`.b`), drawn from `eosio.nft.ft`'s real migrations: the `migration` singleton,
  on-the-fly + batched-bulk migration, dual-write coexistence (`token.c`), the off-chain
  driver, and the activate/gate/retire/rollback rollout sequence.

## Prompt patterns that work

- *"Read `ultra-agent-kb/README.md`, then build and test a contract that …"*
- *"Read `ultra-agent-kb/README.md`, then build a dapp that … using the worked
  example (`09`) as the template."*

## Scope & provenance

This repo is deliberately **standalone**: it contains no credentials and no sensitive
operational data, so it can be shared more widely than Ultra's internal docs. Deep-dive
references into Ultra-internal material (the private `ultraOS-doc` repo, private source
repos) are tagged `[internal]` and are never load-bearing — every doc carries the facts it
needs inline, and `00` gives public alternatives. Repo-relative paths (`eosio.contracts/...`) refer to
checkouts of private Ultra repos; public readers substitute the `00` §3 bootstrap. Content was synthesized from developers.ultra.io, the Ultra source repos, and
shipped wallet/dapp codebases. The contract/dapp/local-chain path was **clean-room-validated**
(2026-07-23) by an agent that built the Tip Jar stack (`09`) from this KB alone. On 2026-09-03 its
wallet layer was updated against the dual-provider patterns in Ultra Bridge and Ultra Tool Kit.
Clean-room validation did not use a real Web Wallet; `06` states the additional release gate.

**Clean-room validated on the public path.** A fresh agent, given only this KB +
`quay.io/ultra.io/3rdparty-devtools:0.4.1-ubuntu24` (re-validated 2026-09-25 with a new escrow
contract + specs + Vue dapp; earlier runs used the July-2026 build) + public npm — every host tool and every other
path on the machine forbidden — designs and ships a complete contract + dapp end to end:
`cdt-cpp` build, ultratest2 spec suite, dapp unit tests, `vue-tsc + vite build`, and
live-chain integration (real signed action through an extension-shaped mock, surfaced contract
assert), then follows doc `08`'s mainnet runbook against live testnet. Provider-selection tests now
cover the Web branch, but a product must also pass `06` §9's real Web Wallet smoke before release.
