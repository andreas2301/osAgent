---
milestone: M2
title: "v0.2-engineer pre-research"
prepared: 2026-06-14
status: ready-for-gsd-new-milestone
---

# M2 — Engineer Binary Production-Ready: Pre-Research

M1 (Foundation) is structurally complete (cargo cleanup in PR #3). M2 scope — per ROADMAP.md and REQUIREMENTS.md — is the engineer binary production-ready milestone. This document synthesizes existing codebase scaffolding, identifies blockers, and scopes cross-cutting dependencies so `/gsd:new-milestone v0.2-engineer` has full context.

## Phase List & Success Criteria (from ROADMAP.md)

M2 is not yet roadmapped in ROADMAP.md (only M1 phases 1.1–1.6 are detailed). However, REQUIREMENTS.md v2 block lists the engineer runtime additions:

- **ENG-BRIDGE**: Native AMQP bridge tool (replaces shell-invoked engineer-amqp-bridge; uses lapin + tokio-rustls with ServerName override for rabbitmq.shield.internal SAN; operator allowlist at /etc/zeroclaw/operator/allowlist.json)
- **ENG-EXCHANGE**: First-class exchange channel with native PLAN/MISSION/REPORT envelope schemas (replaces ad-hoc file-polling per HEARTBEAT.md)
- **ENG-LIFECYCLE**: Pause-marker + activation-marker as daemon primitives; CancellationToken passed into every tool; Vault writes complete current transaction on pause then halt
- **ENG-SQLCIPHER**: Memory backend using rusqlite + bundled-sqlcipher-vendored-openssl; key derived from customer_id + vault-supplied-salt; cross-customer restore fails fast
- **ENG-AUDIT**: Hash-chained dual-sink audit log (journald + append-only file per customer at /var/log/sovereign-shield/osagent-<customer_id>.audit; daily anchor cross-linked to witness chain)
- **ENG-CHANNELS-RT**: Mattermost (in-house reqwest wrapper, v4 REST + WS) + Matrix (matrix-sdk 0.18) channel runtime implementations
- **ENG-CHANNEL-ROLES**: ops vs observer role allowlist per channel
- **ENG-PARITY**: Functional parity audit with current zeroclaw engineer; engineer cutover in install-guide; production deployment

## M2 Requirements (from REQUIREMENTS.md v2 block, lines 57–73)

All M2 requirements: ENG-BRIDGE, ENG-EXCHANGE, ENG-LIFECYCLE, ENG-SQLCIPHER, ENG-AUDIT, ENG-CHANNELS-RT, ENG-CHANNEL-ROLES, ENG-PARITY, MIG-ENG.

**Cross-cutting dependencies on M1**:
- Phase 1.5 cargo cleanup (PR #3) must merge first
- All channel/provider strips + MANIFEST emission from M1 lock down v1 surface
- Wizard-no-MCP 4-layer gate (Phase 1.3) is regression blocker for every commit

## Codebase Scaffolding Audit

### Native Bridge Tool (ENG-BRIDGE)
**Status**: No existing osagent-bridge crate. No lapin or tokio-rustls deps in workspace.
- **File refs**: bins/engineer/Cargo.toml (lines 36–40, currently empty)
- **Blocker**: Must scaffold crates/osagent-bridge/ in M2 Phase 1

### Exchange Channel (ENG-EXCHANGE)
**Status**: No exchange channel implementation in zeroclaw-channels. Signal-cli subprocess pattern exists as precedent.
- **File refs**: crates/zeroclaw-channels/src/lib.rs (lines 3–20 list v1 channels: telegram, slack, matrix, mattermost, whatsapp-cloud, signal, cli — no exchange)
- **Blocker**: Exchange channel must be built from scratch in M2

### Lifecycle Gates (ENG-LIFECYCLE)
**Status**: CancellationToken infrastructure partially in place. Pause semantics exist but not as daemon primitives.
- **File refs**: zeroclaw-runtime/src/agent/loop_.rs (CancellationToken usage), zeroclaw-runtime/src/cron/mod.rs (pause_job function), zeroclaw-runtime/src/heartbeat/engine.rs (paused task status)
- **Gap**: Vault write coordination on pause is new work

### SQLCipher Memory (ENG-SQLCIPHER)
**Status**: zeroclaw-memory uses bundled rusqlite, no sqlcipher-vendored-openssl yet.
- **Current**: zeroclaw-memory/Cargo.toml (line 18) has rusqlite 0.37 with bundled feature. zeroclaw-memory/src/sqlite.rs (lines 21–38) SqliteMemory struct exists but no encryption
- **File refs**: crates/zeroclaw-memory/Cargo.toml (line 18), crates/zeroclaw-memory/src/sqlite.rs (lines 21–80)
- **Blocker**: Must add bundled-sqlcipher-vendored-openssl feature; implement customer-derived key derivation

### Audit Hash-Chain (ENG-AUDIT)
**Status**: Audit infrastructure exists in memory + runtime modules, but hash-chaining + dual-sink + witness anchor not implemented.
- **File refs**: crates/zeroclaw-memory/src/audit.rs (AuditedMemory decorator with SQLite table), crates/zeroclaw-runtime/src/security/audit.rs (exists)
- **Blocker**: Dual-sink (journald + file), hash-chain logic, witness cross-link must be new phases

### Mattermost + Matrix Runtimes (ENG-CHANNELS-RT)
**Status**: Both channels have non-trivial implementations. Mattermost uses reqwest wrapper. Matrix uses matrix-sdk 0.16 (need to bump to 0.18 per spec).
- **File refs**: crates/zeroclaw-channels/src/mattermost.rs (MattermostChannel with REST polling, v4 API, thread replies, bot tokens, allowed_users, mention_only), crates/zeroclaw-channels/src/matrix.rs (matrix-sdk 0.16 with E2EE, sync, encrypted upload), crates/zeroclaw-channels/Cargo.toml (lines 31, 91)
- **Blocker**: Version bump to matrix-sdk 0.18; confirm Mattermost WS support (currently REST polling)

### Signal-cli Precedent (subprocess pattern)
**Status**: Signal channel already uses signal-cli subprocess with JSON-RPC over HTTP + SSE.
- **File refs**: crates/zeroclaw-channels/src/signal.rs, crates/zeroclaw-channels/src/lib.rs (line 9)
- **Takeaway**: Subprocess pattern established; not a blocker for M2

## PROJECT.md Blockers & Unresolved Decisions

All 42 ratified decisions are constraints. M2-relevant: #1 (two binaries), #8 (idempotency keys), #9 (sqlcipher), #21 (pause semantics), #22 (dual-sink audit), #39 (witness anchor), #41 (telegram bots).

**No design regressions found.** All decisions are constraints, not open questions.

## bins/engineer/Cargo.toml Placeholder Status

Current: dependencies block is empty. Comments name M2 work: osagent-runtime, osagent-channels, osagent-tools, osagent-bridge (NEW), osagent-exchange (NEW), osagent-lifecycle (NEW or extend runtime), osagent-audit (NEW or extend memory), osagent-tools-mcp (shared, engineer-only).

**Blocker list for M2 kickoff**:
1. Scaffold crates/osagent-bridge/ (lapin + tokio-rustls; operator allowlist)
2. Add exchange channel to crates/zeroclaw-channels/ or create crates/osagent-exchange/
3. Create or extend crates/osagent-audit/ for dual-sink + hash-chain
4. Possibly new crates/osagent-lifecycle/ if logic substantial
5. Add all deps to bins/engineer/Cargo.toml with default-features=false + explicit features

## Summary for `/gsd:new-milestone v0.2-engineer`

**Ready state**: M1 structural gates complete. PR #3 (cargo cleanup) is final M1 blocker.

**M2 scope**: 8 features across bridge, exchange, lifecycle, sqlcipher, audit, channels, roles, parity.

**Crate scaffolding needed**: osagent-bridge (net-new), osagent-exchange or zeroclaw-channels extension, osagent-audit or zeroclaw-memory extension.

**Version bumps**: matrix-sdk 0.16 → 0.18.

**No architectural regressions**: All 42 ratified decisions are constraints.

**Install-guide tie-in**: PR feat/osagent-install-task (M1-opened) awaits M2 completion for merge (meta:end_play guard in place).

---

*Pre-research prepared 2026-06-14. Ready for `/gsd:new-milestone v0.2-engineer` execution.*
