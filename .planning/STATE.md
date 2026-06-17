# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-06-17)

**Core value:** Wizard cannot exfiltrate Vault secrets via an MCP server because the wizard binary has compile-time zero MCP, verified by a 4-layer CI gate (source-grep + `nm --defined-only` + `cargo-bloat --crates` + `strings`).
**Current focus:** M2 (v0.2-engineer) — roadmap complete (phases 2.1–2.9); Phase 2.1 (Native AMQP Bridge + MANIFEST build.rs emission) is the next executable phase. M1 closed 2026-06-17 (PR #3 merged structural cargo cleanup; PR #5 doc recovery + M2 pre-research stacked, awaiting merge).

## Current Position

Phase: 2.1 — Native AMQP Bridge + MANIFEST build.rs emission (not started)
Plan: — (planning has not begun; next step: `/gsd:plan-phase 2.1` or `/gsd:discuss-phase 2.1`)
Status: M2 roadmap created (9 phases, 32/32 requirements mapped, 0 orphans). Ready for first-phase planning.
Last activity: 2026-06-17 — M2 roadmap appended to ROADMAP.md; REQUIREMENTS.md traceability filled

Progress: M2 [░░░░░░░░░░] 0% (0/9 phases shipped)

## Accumulated Context

### Decisions

The 42 ratified pre-init decisions in PROJECT.md remain authoritative. M1 added three cross-cutting refinements (preserved):

- **Refinement #1 (Decision #1):** MCP boundary is structural crate exclusion (`osagent-tools-mcp` separate crate, wizard has no dependency edge), NOT Cargo features. Defeats resolver=2 feature unification.
- **Refinement #2 (Decision #25):** CI gate is 4-layer (source-grep + `nm --defined-only` + `cargo bloat --crates` + `strings`), NOT single `nm` grep. LTO/strip/DCE defeats single-layer.
- **Refinement #3:** Signal channel is out-of-process `signal-cli` subprocess (M4); cargo-deny ban entries for AGPL Rust Signal SDKs (presage, libsignal*) landed in M1 Phase 1.1.

M2 roadmap adds three structural composition decisions (not new design, just placement):

- **Composition #1:** MANIFEST-04 (build.rs emission) folds into Phase 2.1 alongside ENG-BRIDGE. Build.rs is independent of runtime feature code so it can land first; the manifest-equality gate then catches "did this strip orphan a feature?" for every later phase.
- **Composition #2:** Gateway sub-surface coherent trim (M1 closeout carry-over) folds into Phase 2.6 alongside ENG-CHANNELS-RT + ENG-CHANNEL-ROLES. Engineer wiring to the surviving `/ws/chat` + `paired_tokens` + node discovery surfaces is the natural point to trim dead canvas/sse/acp/static_files/api_pairing references and remove the `CanvasStore` stub.
- **Composition #3:** Phase 2.2 (SQLCipher memory) ∥ Phase 2.3 (audit hash-chain) parallel-eligible per M2-PRE-RESEARCH constraint #2; both block Phase 2.4 (lifecycle, which needs encrypted memory + audit log to pause-mid-transaction-test against).

New M2-specific implementation decisions get logged at phase-transition time.

### M2 Pre-Research

Synthesis at `.planning/M2-PRE-RESEARCH.md` (105 lines, prepared 2026-06-14). Key findings:

- 8 engineer features confirmed (ENG-BRIDGE, ENG-EXCHANGE, ENG-LIFECYCLE, ENG-SQLCIPHER, ENG-AUDIT, ENG-CHANNELS-RT, ENG-CHANNEL-ROLES, ENG-PARITY) from REQUIREMENTS.md v2 block, plus MIG-ENG and the two deferred MANIFEST tasks.
- **Scaffolding state per feature** with file:line refs in M2-PRE-RESEARCH.md.
- **Net-new work:** ENG-BRIDGE crate (no `lapin`/`tokio-rustls` yet), exchange channel, SQLCipher feature on `rusqlite`, dual-sink + hash-chain audit.
- **Partial scaffolding:** lifecycle gates (~60% — `CancellationToken` infrastructure exists in zeroclaw-runtime/src/agent; Vault transaction coordination on pause is new), AuditedMemory base exists (audit.rs).
- **Version bumps required:** `matrix-sdk` 0.16 → 0.18 (and confirm mattermost WS support route).
- **No architectural regressions** — all 42 ratified decisions remain valid constraints.

### M2 Roadmap

Created 2026-06-17. 9 phases (2.1 → 2.9). 32/32 v1 requirements mapped, 0 orphans.

| Phase | Goal (one-line) | Requirements |
|-------|-----------------|--------------|
| 2.1 | Native AMQP bridge + MANIFEST build.rs emission | ENG-BRIDGE-01..04, MANIFEST-04 |
| 2.2 | SQLCipher memory (customer-derived key) | ENG-SQLCIPHER-01..04 |
| 2.3 | Audit hash-chain dual-sink (journald + per-customer file) | ENG-AUDIT-01..04 |
| 2.4 | Lifecycle gates (pause-marker + CancellationToken + Vault-write atomicity) | ENG-LIFECYCLE-01..04 |
| 2.5 | Exchange channel (typed PLAN/MISSION/REPORT) | ENG-EXCHANGE-01..03 |
| 2.6 | Mattermost+Matrix runtime + channel roles + gateway sub-surface coherent trim | ENG-CHANNELS-RT-01..03, ENG-CHANNEL-ROLES-01..03 |
| 2.7 | `manifest --diff` CLI refuse-to-start | MANIFEST-05 |
| 2.8 | Parity audit + clean-VM smoke + upgrade-in-place smoke | ENG-PARITY-01..03 |
| 2.9 | Engineer migration cutover (install-guide PR merge + reference swap + binary purge) | MIG-ENG-01..02 |

Execution order: 2.1 → (2.2 ∥ 2.3) → 2.4; in parallel 2.1 → (2.5 ∥ 2.6); 2.1 → 2.7; everything → 2.8 → 2.9.

### Adopted in-flight (carried from M1)

- **TDD discipline ratified.** Bash test suite + Rust integration tests; M2 continues the contract. 332/332 assertions green at M1 closeout.
- **Branch protection on `osagent-main`.** All 7-8 CI status checks required (self-approval guard active — author cannot merge own PR).
- **PR open via API only.** Per UPSTREAM_SYNC.md Tooling Discipline section: always explicit `base`, verify `head.repo == base.repo`, never use git-push URL hints.
- **Plan-then-execute hard rule on install-guide work.** Per `sovereign-shield-install-guide/CLAUDE.md`: Touch/Change/Impact/Rollback plan BEFORE any file edit. Overrides auto-mode for that repo. Phase 2.9 is the natural touch-point.

### Pending Todos

1. **PR #5 (osAgent)** — `docs/m1-audit-recovery-plus-m2-research` — flips M1 audit doc cargo-red → green, commits M2 pre-research onto `osagent-main`. Awaiting user merge. Phase 2.1 does NOT block on this (works from `gsd/v0.2-engineer-init` via recovery chain).
2. **PR #10 (sovereign-shield-backup)** — `feat/osagent-upstream-sync-runbook` — UPSTREAM_SYNC.md + Tooling Discipline section + stacked-PR correction. Awaiting user merge.
3. **PR #141 (sovereign-shield-install-guide)** — `feat/osagent-install-task` — `install_osagent.yml` structural template with `meta:end_play` guard. Hold-merge-until-M2-completes — Phase 2.9 is the merge trigger.

### Blockers/Concerns

- **PR #5 not yet merged** — M2-PRE-RESEARCH.md still off `osagent-main`. Working in `gsd/v0.2-engineer-init` branch which has it via the recovery chain; future M2 phase work that cuts from `osagent-main` will not see the pre-research until PR #5 merges. Phase 2.1 should branch from `gsd/v0.2-engineer-init` if PR #5 has not landed.
- **No cargo locally** — CI-iteration loop pattern (from UPSTREAM_SYNC.md M1 lessons) still applies for every M2 build/test cycle.

## Session Continuity

Last session: 2026-06-17 — M2 roadmap created via `/gsd:roadmap` (this run)
Stopped at: ROADMAP.md M2 section appended (phases 2.1–2.9); REQUIREMENTS.md traceability filled; STATE.md current position set to Phase 2.1
Resume file: `.planning/ROADMAP.md` (M2 section), `.planning/REQUIREMENTS.md` (M2 v1 block + traceability), `.planning/M2-PRE-RESEARCH.md` (scaffolding audit), this STATE.md

## Next session

Run order for a `/clear` + resume on M2:
1. Read `PROJECT.md` (Current Milestone section), `REQUIREMENTS.md` (M2 v1 block + Traceability), `ROADMAP.md` (M2 phase details starting at "### Phase 2.1"), `M2-PRE-RESEARCH.md` (scaffolding audit with file:line refs), this `STATE.md`.
2. `cd d:/Repositories/osAgent && git checkout gsd/v0.2-engineer-init` (or `osagent-main` if PR #5 has merged).
3. `/gsd:plan-phase 2.1` (or `/gsd:discuss-phase 2.1` for context-gathering first).
4. Continue per usual GSD discipline (TDD, atomic commits, PR via API with explicit base, monitor CI).
