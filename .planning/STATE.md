# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-06-17)

**Core value:** Wizard cannot exfiltrate Vault secrets via an MCP server because the wizard binary has compile-time zero MCP, verified by a 4-layer CI gate (source-grep + `nm --defined-only` + `cargo-bloat --crates` + `strings`).
**Current focus:** M2 (v0.2-engineer) — defining requirements + roadmap. M1 closed 2026-06-17 (PR #3 merged structural cargo cleanup; PR #5 doc recovery + M2 pre-research stacked, awaiting merge).

## Current Position

Phase: Not started (defining requirements + roadmap for M2)
Plan: —
Status: Milestone v0.2-engineer initialized — requirements drafted, roadmap pending
Last activity: 2026-06-17 — M1 milestone closed, M2 milestone started

Progress: M2 [░░░░░░░░░░] 0%

## Accumulated Context

### Decisions

The 42 ratified pre-init decisions in PROJECT.md remain authoritative. M1 added three cross-cutting refinements (preserved):

- **Refinement #1 (Decision #1):** MCP boundary is structural crate exclusion (`osagent-tools-mcp` separate crate, wizard has no dependency edge), NOT Cargo features. Defeats resolver=2 feature unification.
- **Refinement #2 (Decision #25):** CI gate is 4-layer (source-grep + `nm --defined-only` + `cargo bloat --crates` + `strings`), NOT single `nm` grep. LTO/strip/DCE defeats single-layer.
- **Refinement #3:** Signal channel is out-of-process `signal-cli` subprocess (M4); cargo-deny ban entries for AGPL Rust Signal SDKs (presage, libsignal*) landed in M1 Phase 1.1.

M2 carries forward without re-litigating any of these. New M2-specific implementation decisions get logged at phase-transition time.

### M2 Pre-Research

Synthesis at `.planning/M2-PRE-RESEARCH.md` (105 lines, prepared 2026-06-14). Key findings:

- 8 engineer features confirmed (ENG-BRIDGE, ENG-EXCHANGE, ENG-LIFECYCLE, ENG-SQLCIPHER, ENG-AUDIT, ENG-CHANNELS-RT, ENG-CHANNEL-ROLES, ENG-PARITY) from REQUIREMENTS.md v2 block, plus MIG-ENG and the two deferred MANIFEST tasks.
- **Scaffolding state per feature** with file:line refs in M2-PRE-RESEARCH.md.
- **Net-new work:** ENG-BRIDGE crate (no `lapin`/`tokio-rustls` yet), exchange channel, SQLCipher feature on `rusqlite`, dual-sink + hash-chain audit.
- **Partial scaffolding:** lifecycle gates (~60% — `CancellationToken` infrastructure exists in zeroclaw-runtime/src/agent; Vault transaction coordination on pause is new), AuditedMemory base exists (audit.rs).
- **Version bumps required:** `matrix-sdk` 0.16 → 0.18 (and confirm mattermost WS support route).
- **No architectural regressions** — all 42 ratified decisions remain valid constraints.

### Adopted in-flight (carried from M1)

- **TDD discipline ratified.** Bash test suite + Rust integration tests; M2 continues the contract. 332/332 assertions green at M1 closeout.
- **Branch protection on `osagent-main`.** All 7-8 CI status checks required (self-approval guard active — author cannot merge own PR).
- **PR open via API only.** Per UPSTREAM_SYNC.md Tooling Discipline section: always explicit `base`, verify `head.repo == base.repo`, never use git-push URL hints.

### Pending Todos

1. **PR #5 (osAgent)** — `docs/m1-audit-recovery-plus-m2-research` — flips M1 audit doc cargo-red → green, commits M2 pre-research onto `osagent-main`. Awaiting user merge.
2. **PR #10 (sovereign-shield-backup)** — `feat/osagent-upstream-sync-runbook` — UPSTREAM_SYNC.md + Tooling Discipline section + stacked-PR correction. Awaiting user merge.
3. **PR #141 (sovereign-shield-install-guide)** — `feat/osagent-install-task` — `install_osagent.yml` structural template with `meta:end_play` guard. Hold-merge-until-M2-completes (ENG-PARITY locks the binary URL + SHA256 before this can flip to a live install).

### Blockers/Concerns

- **PR #5 not yet merged** — M2-PRE-RESEARCH.md still off `osagent-main`. Working in `gsd/v0.2-engineer-init` branch which has it via the recovery chain; future M2 phase work that cuts from `osagent-main` will not see the pre-research until PR #5 merges.
- **No cargo locally** — CI-iteration loop pattern (from UPSTREAM_SYNC.md M1 lessons) still applies for every M2 build/test cycle.

## Session Continuity

Last session: 2026-06-17 — M1 closed; M2 initialized via `/gsd:new-milestone v0.2-engineer`
Stopped at: PROJECT.md + STATE.md updated for M2; REQUIREMENTS.md update + ROADMAP.md generation in flight on `gsd/v0.2-engineer-init` branch
Resume file: `.planning/PROJECT.md` (current milestone section), `.planning/M2-PRE-RESEARCH.md` (synthesis), this STATE.md

## Next session

Run order for a `/clear` + resume on M2:
1. Read `PROJECT.md` (Current Milestone section), `REQUIREMENTS.md` (M2 v1 block), `M2-PRE-RESEARCH.md` (scaffolding audit), `ROADMAP.md` (phase plan), this `STATE.md`.
2. `cd d:/Repositories/osAgent && git checkout gsd/v0.2-engineer-init` (or `osagent-main` if merged).
3. Pick the first phase from `ROADMAP.md` — typically `/gsd:discuss-phase 2` for context-gathering or `/gsd:plan-phase 2` to skip discussion.
4. Continue per usual GSD discipline (TDD, atomic commits, PR via API with explicit base, monitor CI).
