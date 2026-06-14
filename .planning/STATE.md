# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-06-12)

**Core value:** Wizard cannot exfiltrate Vault secrets via an MCP server because the wizard binary has compile-time zero MCP, verified by a 4-layer CI gate (source-grep + `nm --defined-only` + `cargo-bloat --crates` + `strings`).
**Current focus:** M1 structurally complete; cargo cleanup in PR `chore/01.5-cargo-cleanup` awaiting CI; bookkeeping synced 2026-06-13.

## Current Position

Phase: 1.6 of 1.6 structurally complete; Phase 1.5 partial (cargo cleanup deferred to PR)
Plan: M1 milestone audit shipped at `.planning/v1-M1-MILESTONE-AUDIT.md`
Status: 5 of 6 phases ✅ complete; 1 partial (1.5); milestone-defining gate (Phase 1.3) green; 332 test assertions green
Last activity: 2026-06-13 — Bookkeeping sync after 14+ commits; branch protection verified; cargo cleanup PR opened

Progress: [█████████░] 90% (M1 structural; cargo-green follow-up pending)

## Performance Metrics

**Velocity:**
- Phases complete: 5 of 6 structurally
- Phase artifacts shipped: 6 CONTEXT, 4 RESEARCH, 1 VALIDATION, 6 SUMMARY, 6 VERIFICATION (after this commit)
- Test assertions: 332 bash + 16 Rust integration

**By Phase:**

| Phase | Status | Tests | Notes |
|-------|--------|-------|-------|
| 1.1 Fork & Attribution & Sync Runbook | ✅ | 46 | Public fork + UPSTREAM_SYNC.md PR on sovereign-shield-backup |
| 1.2 Workspace Skeleton & Binary Split | ✅ | 30 | bins/engineer + bins/wizard; WS-04 distributed-slice ban |
| 1.3 MCP Boundary & 4-Layer CI Gate | ✅ MILESTONE-DEFINING | 18 | osagent-tools-mcp crate; 4-layer wizard gate green |
| 1.4 Whole-Crate Drops & Telemetry | ✅ | 48 | ~16.2K LOC removed; OTLP stripped; MCP files migrated |
| 1.5 Source Strips & MANIFEST | 🟡 partial | 156 | 28 channels + 9 providers + 36 tools + non-en locales stripped; osagent-manifest crate full TDD; cascading cargo cleanup in PR `chore/01.5-cargo-cleanup` |
| 1.6 Gateway Fork & Install Drop-In | ✅ | 34 | gateway sub-surface stripped; install_osagent.yml PR on sovereign-shield-install-guide |

**Recent Trend:**
- Last 5 commits on osagent-main: bookkeeping → milestone-audit → 1.6 → 1.5 cleanup → 1.5 TDD baseline
- Trend: ↑ (structural foundation complete; momentum into M2 once cargo cleanup PR merges)

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table (42 ratified pre-init decisions).
Cross-cutting refinements applied at roadmap creation:

- **Refinement #1 (Decision #1):** MCP boundary is structural crate exclusion (`osagent-tools-mcp` separate crate, wizard has no dependency edge), NOT Cargo features. Defeats resolver=2 feature unification.
- **Refinement #2 (Decision #25):** CI gate is 4-layer (source-grep + `nm --defined-only` + `cargo bloat --crates` + `strings`), NOT single `nm` grep. LTO/strip/DCE defeats single-layer.
- **Refinement #3:** Signal channel is out-of-process `signal-cli` subprocess (M4); cargo-deny ban entries for AGPL Rust Signal SDKs (presage, libsignal*) landed in M1 Phase 1.1.

### Adopted in-flight

- **TDD discipline ratified mid-M1.** Every new feature ships test-first; bash test suite + Rust integration tests serve as the contract. 332 assertions / 6 suites / 0 failures locally.
- **Branch protection on `osagent-main` active.** All 7 CI status checks required: cargo-deny (licenses+bans+sources), cargo-deny (advisories), WS-04 no-distributed-slice, workspace build, wizard-no-mcp-gate (4-layer), tests/ bash suite, cargo test (engineer + wizard). Verified by rejected direct push.

### Pending Todos

1. **Cargo cleanup PR** at `chore/01.5-cargo-cleanup` — first pass already committed (~6 dropped tool registrations + browser stack block removed). CI will surface remaining ~17 Arc::new(<DroppedTool>::new(...)) call sites that need removal. Best resolved in a focused session with `cargo check --workspace` running locally.
2. **Three PRs awaiting user merge**:
   - sovereign-shield-backup ← `feat/osagent-upstream-sync-runbook` (UPSTREAM_SYNC.md)
   - sovereign-shield-install-guide ← `feat/osagent-install-task` (install_osagent.yml structural template)
   - osAgent ← `chore/01.5-cargo-cleanup` (after CI iteration)
3. **build.rs MANIFEST emission** — `osagent-manifest` crate ships; the build.rs that writes `[declared]+[detected]` MANIFEST.toml + binary --manifest-diff CLI wiring is the remaining Phase 1.5 work.

### Blockers/Concerns

- Cargo cleanup is best-done with cargo locally; blind iteration via CI round-trips is inefficient.
- M2 planning blocked on cargo-green M1 (so the engineer runtime work can build against a known-good baseline).

## Session Continuity

Last session: 2026-06-13
Stopped at: GSD bookkeeping synced after 14+ commits of structural M1 work; cargo cleanup PR awaiting focused follow-up session
Resume file: `.planning/v1-M1-MILESTONE-AUDIT.md` (canonical milestone status); this STATE.md (bookkeeping mirror)

## Next session

Run order for a `/clear` + resume:
1. Read `PROJECT.md`, `REQUIREMENTS.md`, this `STATE.md`, `.planning/v1-M1-MILESTONE-AUDIT.md`.
2. `cd d:/Repositories/osAgent && git checkout chore/01.5-cargo-cleanup`.
3. `cargo check --workspace` (must have cargo locally for fast feedback).
4. Iterate against the cascading-cleanup todo list from the milestone audit until green.
5. Push fixes; CI on `chore/01.5-cargo-cleanup` PR should go all-7 green.
6. Merge the PR (branch protection enforces all 7 checks).
7. Either: start M2 via `/gsd:new-milestone v0.2-engineer` OR finish `build.rs` MANIFEST emission first.
