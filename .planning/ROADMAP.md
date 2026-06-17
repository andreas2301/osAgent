# Roadmap: osAgent — M1 (Foundation)

## Overview

osAgent is a tailored fork of zeroclaw v0.7.5 producing two compile-time-separated binaries (`engineer` and `wizard`) that drop into sovereign-shield's existing systemd units. **M1 — Foundation** ships a buildable, provably-shaped fork: public repo with attribution, two-binary workspace topology, structural MCP exclusion verified by a 4-layer CI gate, ~60K LOC of dead surface deleted (channels, providers, tools, gateway sub-surface, webhooks, telemetry), a build-time `MANIFEST.toml` that doesn't lie, and a drop-in ansible install task for the engineer binary. M1 is about **shape, not behavior** — engineer/wizard runtime feature changes (native AMQP bridge, sqlcipher memory, hash-chain audit, 2-person Vault approval, subagents, full channel runtimes) land in M2/M3/M4. The milestone-defining green check is Phase 1.3: the 4-layer wizard-no-MCP CI gate passing on every PR. The riskiest work is Phase 1.5 (24 named pitfalls concentrated there). The slow-burn risk is Phase 1.1 (fork rots without UPSTREAM_SYNC.md discipline).

## Milestones

- ✅ **M1 — Foundation** — Phases 1.1–1.6 shipped 2026-06-17 (5 structural complete; 1 partial — MANIFEST `build.rs` emission + binary `--manifest-diff` wiring deferred to M2 as MANIFEST-04 + MANIFEST-05)
- 🚧 **M2 — Engineer binary production-ready** — Phases 2.1–2.9 roadmapped 2026-06-17
- 📋 **M3 — Wizard binary + subagent system** — planned, not yet roadmapped
- 📋 **M4 — Channels + Ops + Provider routing + Production rollout** — planned, not yet roadmapped

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (1.1, 1.2, ...): M1's six foundation sub-phases (executed in numeric order)
- Decimal phases (2.1, 2.2, ...): M2's nine engineer-runtime sub-phases (executed in numeric order; some parallel-eligible — see Progress section)

**Bookkeeping note**: M1 phases were executed inline (CONTEXT → execute → SUMMARY/VERIFICATION) without per-phase `/gsd:transition` calls, which is why all artifacts ship on disk but this checkbox list lagged. Synced 2026-06-13.

### M1 — Foundation (shipped)

- [x] **Phase 1.1: Fork & Attribution & Sync Runbook** ✅ (commit fc3139a, 8165015) — Public `andreas2301/osAgent` fork live, working branch `osagent-main`, NOTICE preserved + osAgent attribution + Apache-2.0 §4(d) verbatim block, `deny.toml` with AGPL/WTFPL/phone-home bans, SHA-pinned cargo-deny-action CI, UPSTREAM_SYNC.md runbook on `feat/osagent-upstream-sync-runbook` (sovereign-shield-backup PR pending merge). **46/46 tests green.**
- [x] **Phase 1.2: Workspace Skeleton & Binary Split** ✅ (commit e614cee) — `bins/engineer` + `bins/wizard` workspace members; resolver=2 already pinned by upstream; WS-04 ratified (no `inventory!`/`linkme!`/`ctor!` in source; `deny.toml` bans the crates; CI `no-distributed-slice-registration` job); workspace-build CI job. **30/30 tests green.**
- [x] **Phase 1.3: MCP Boundary & 4-Layer CI Gate** ✅ MILESTONE-DEFINING (commit e332aa1) — `crates/osagent-tools-mcp/` workspace member; `bins/wizard/Cargo.toml` has zero MCP dep declaration; 4-layer gate script (`scripts/wizard-no-mcp-gate.sh`); CI job `wizard-no-mcp-gate` needs:[workspace-build], installs `cargo-bloat`, runs all 4 layers. **18/18 tests green; layer 1 verified locally; layers 2-4 fire on CI Linux.**
- [x] **Phase 1.4: Whole-Crate Drops & Telemetry Audit** ✅ (commit 07dde29) — ~16.2K LOC deleted: `zeroclaw-hardware` (9.5K), `robot-kit` (3.5K), `aardvark-sys` (0.5K), `apps/tauri` (0.8K), `zeroclaw-plugins` (1.9K). Webhook channel triple-removed. `opentelemetry-otlp` + `observability-otel` removed. MCP files physically migrated to `osagent-tools-mcp`. **48/48 tests green.**
- [~] **Phase 1.5: Source Strips & MANIFEST Emission** 🟡 PARTIAL (commits 0a9da02, beec3af, 9d3e2fd) — **Structural strips ✅**: 28 channel sources removed (v1 kept: telegram, slack, matrix, mattermost, whatsapp-cloud, signal); 9 provider sources removed (v1 kept: anthropic, gemini, openai-compatible base, ollama, openrouter); 36 tool sources removed; non-en locales removed (Fluent pipeline kept); `osagent-manifest` crate ships with 8 TDD tests (RED → GREEN); MANIFEST.toml.template scaffold; reproducibility profile verified. **156/156 bash tests green.** **PARTIAL** = `build.rs` MANIFEST emission + binary `--manifest-diff` wiring rolled forward to M2 Phase 2.1 (MANIFEST-04) + Phase 2.7 (MANIFEST-05).
- [x] **Phase 1.6: Gateway Fork & Install Drop-In** ✅ (commit 269b050) — Gateway sub-surface stripped (ACP/REST endpoints/SSE/static-files/openapi/tls/voice/ws_approval); `/ws/chat` + `paired_tokens` kept (OS-MDashboard chat-relay dependency); install-guide PR `feat/osagent-install-task` on sovereign-shield-install-guide with structural `install_osagent.yml` template (`meta:end_play` guard prevents accidental production deploy until M2 fills binary URL + SHA256). **34/34 tests green.**

**Milestone-level extras shipped during M1**:
- TDD discipline framework (`tests/lib.sh`, `tests/run-all.sh`, per-phase test files) — **332 bash assertions / 6 suites / 0 failures locally**
- Rust integration tests in `bins/{engineer,wizard}/tests/binary_smoke.rs` + `crates/osagent-manifest/tests/manifest_diff.rs` (16 Rust tests; CI runs them)
- 7-job CI workflow at `.github/workflows/osagent-policy.yml` with branch protection on `osagent-main` (all 7 required for merge; verified by rejected direct push 2026-06-13)
- Full milestone audit at [`.planning/v1-M1-MILESTONE-AUDIT.md`](v1-M1-MILESTONE-AUDIT.md)

### M2 — v0.2-engineer (Engineer binary production-ready)

**Milestone overview.** M1 shipped the *shape* (two-binary workspace, MCP-excluded wizard, dead-surface deleted). M2 fills the engineer binary with production *behavior*: a native AMQP bridge (replaces bash+python3+shell chain), encrypted memory, hash-chained audit, lifecycle gates that don't tear Vault transactions, a typed PLAN/MISSION/REPORT exchange channel, Mattermost+Matrix runtime wiring with `ops`/`observer` role enforcement, a build-time `MANIFEST.toml` that doesn't lie (deferred from M1), and `osagent engineer manifest --diff` refuse-to-start CLI. Phase 2.1 (bridge) is the foundation — every later phase depends on it directly or transitively, because "engineer" without operator-allowlisted AMQP isn't engineer. Phases 2.2 (SQLCipher memory) and 2.3 (audit hash-chain) are parallel-eligible after 2.1 — both touch persistent storage independently and both are prerequisites for 2.4 (lifecycle: pause-mid-transaction tests need encrypted memory + audit log working) and 2.8 (parity audit needs the full storage trio). Phase 2.5 (exchange channel) and Phase 2.6 (Mattermost+Matrix runtime + roles + gateway sub-surface coherent trim) are also parallel-eligible after 2.1 — exchange shares the AMQP infrastructure but channels are an independent crate. Phase 2.7 (`manifest --diff` CLI) folds MANIFEST-05 in once 2.1's `main.rs` subcommand routing exists and MANIFEST-04 (in 2.1) has emitted the file. Phase 2.8 (parity audit + clean-VM + upgrade-in-place smoke tests) gates Phase 2.9 (install-guide cutover, old-binary purge). The riskiest phase is 2.1 (lapin + tokio-rustls SAN-override + fail-closed allowlist + replacing the existing shell chain without parity gap). The slow-burn risk is 2.6 (gateway sub-surface coherent trim while wiring engineer to surviving `/ws/chat` + `paired_tokens`).

- [ ] **Phase 2.1: Native AMQP Bridge + MANIFEST build.rs emission** — Stand up `osagent-bridge` crate (lapin 0.5 + tokio-rustls 0.27 with `rabbitmq.shield.internal` SAN override, dial 127.0.0.1), wire `bridge_call` into engineer-only tool registry, load + enforce 45-verb operator allowlist fail-closed, retire shell+python3 bridge chain, emit per-binary `MANIFEST-<binary>.toml` from `build.rs` with `[declared]` + `[detected]` and a CI equality gate.
- [ ] **Phase 2.2: SQLCipher Memory** — Rebuild `zeroclaw-memory` SQLite backend on `rusqlite` + `bundled-sqlcipher-vendored-openssl`, derive encryption key from `customer_id` + Vault-supplied salt (KDF documented), prove cross-customer restore fails fast (SQLITE_NOTADB), ship ansible re-encrypt migration step for existing plaintext memory.
- [ ] **Phase 2.3: Audit Hash-Chain Dual-Sink** — Wire `tracing-journald` + append-only per-customer file `/var/log/sovereign-shield/osagent-<customer_id>.audit`, sha256 `prev_hash` chain (genesis = 64x"0"), `osagent-witness-anchor` daily cron cross-link to ola-management-witness, `osagent engineer audit verify` CLI with seeded-tamper integration test.
- [ ] **Phase 2.4: Lifecycle Gates** — Daemon-level pause-marker + activation-marker files, `CancellationToken` plumbed into every tool's `execute` via existing zeroclaw-runtime agent extension points, mid-transaction Vault writes complete current idempotency-keyed transaction then halt cleanly, heartbeat + cron respect pause-marker.
- [ ] **Phase 2.5: Exchange Channel** — First-class `exchange` channel in `zeroclaw-channels` with typed PLAN/MISSION/REPORT Rust envelope schemas, durable per-customer RMQ queues, malformed-envelope dead-letter routing, retire HEARTBEAT.md `.plan`/`.mission`/`.report` file-polling.
- [ ] **Phase 2.6: Mattermost + Matrix Runtime + Channel Roles + Gateway Trim** — Mattermost v4 REST + WebSocket in-house `reqwest` wrapper, `matrix-sdk` 0.16 → 0.18 bump with E2EE preserved, both wired to existing M1 `AskUserTool` channel-map handles, `ops`/`observer` role allowlist enforced at tool-dispatch (not LLM-prompt) with deny-by-default + audit log of denied attempts; opportunistic coherent trim of gateway sub-surface dead references (canvas::, sse::, acp::, static_files::, api_pairing::, api_webauthn::) while wiring engineer to surviving `/ws/chat` + `paired_tokens` + node discovery; remove `CanvasStore` stub.
- [ ] **Phase 2.7: MANIFEST `--diff` CLI Wiring** — Wire `osagent engineer manifest --diff <config.toml>` subcommand in engineer's `main.rs` (subcommand router introduced in 2.1) — reads emitted MANIFEST, validates every channel/provider/tool in config exists in binary's manifest, refuse-to-start (exit 1) on bidirectional mismatch.
- [ ] **Phase 2.8: Engineer Parity Audit + Smoke Tests** — Write `.planning/M2-PARITY-AUDIT.md` enumerating 45-verb coverage + HEARTBEAT.md behaviors + scheduled-job semantics + life-config integration points; status ✅/⚠️/❌ per item; zero ⚠️ before 2.9 fires; clean-VM smoke test (Ubuntu 22.04 + `install_osagent.yml` + all 45 verbs + journald/file/SQLCipher assertions); upgrade-in-place smoke test (zeroclaw → osAgent migration on already-deployed VM, no data loss, zeroclaw service stopped + binary removed).
- [ ] **Phase 2.9: Engineer Migration Cutover** — Set binary URL + SHA256 in `install_osagent.yml`, remove `meta:end_play` guard, merge `feat/osagent-install-task` PR on sovereign-shield-install-guide; swap playbook reference `install_zeroclaw.yml` → `install_osagent.yml` (delete old reference, not comment); purge old zeroclaw engineer binary symlink `/usr/local/bin/zeroclaw` on next ansible apply.

## Phase Details

### Phase 1.1: Fork & Attribution & Sync Runbook
**Goal**: Establish a legally-distributable, quarterly-sustainable fork with license discipline ratified at the CI layer.
**Depends on**: Nothing (first phase)
**Requirements**: FORK-01, FORK-02, FORK-03
**Success Criteria** (what must be TRUE):
  1. `andreas2301/osAgent` is public on GitHub, working branch is `osagent-main`, `git fetch upstream` works, `LICENSE-APACHE` + `LICENSE-MIT` + upstream `NOTICE` + osAgent `NOTICE` all present in repo root
  2. `sovereign-shield-backup/documentation/osAgent/UPSTREAM_SYNC.md` exists and documents: quarterly cadence (Q1/Q2/Q3/Q4 first-week), diff-stat budget per merge, conflict-resolution log format, out-of-cycle critical-security-fix criteria, append-only conflict log
  3. `cargo deny check` passes in CI on every PR with license allowlist (MIT/Apache-2.0/BSD-2/BSD-3/ISC/Unicode-DFS-2016) and explicit bans for AGPL-3.0 crates (presage, libsignal-service, libsignal, libsignal-protocol, libsignal-client, libsignal-bridge) and WTFPL (frankenstein)
  4. Advisory database currency check (`cargo deny check advisories`) runs in CI and blocks merge on RUSTSEC matches
**Plans**: 4 plans
- [ ] 01.1-00-PLAN.md — Wave 0 preflight (tooling install, namespace-type discovery, phone-home grep, validation harness skeletons)
- [ ] 01.1-01-PLAN.md — GitHub fork + remotes + osagent-main + NOTICE + Cargo.toml metadata + branch protection [FORK-01]
- [ ] 01.1-02-PLAN.md — deny.toml + .github/workflows/ci.yml (SHA-pinned cargo-deny-action) + required-status-checks update [FORK-03]
- [ ] 01.1-03-PLAN.md — UPSTREAM_SYNC.md runbook on sovereign-shield-backup with Touch/Change/Impact/Rollback PR [FORK-02]

Notes:
- Slow-burn failure mode: without UPSTREAM_SYNC.md as binding ritual, the fork drifts within a quarter. Runbook discipline is what makes Q4 not catastrophic.
- The Signal SDK ban entries are M1-relevant (cargo-deny enforcement) even though Signal channel runtime is M4.

### Phase 1.2: Workspace Skeleton & Binary Split
**Goal**: Establish the two-binary workspace topology and explicit-registration pattern that all subsequent strips depend on.
**Depends on**: Phase 1.1
**Requirements**: WS-01, WS-04, WS-05
**Success Criteria** (what must be TRUE):
  1. `cargo build --workspace` succeeds with `resolver = "2"` pinned and `bins/osagent-engineer/` + `bins/osagent-wizard/` as top-level binary crates whose `Cargo.toml` files are the human-readable manifest of compiled-in capabilities
  2. Channels, providers, and tools are registered via explicit `registry.register(Box::new(Factory))` calls in each binary's `main.rs` — zero use of `inventory::submit!`, `linkme::distributed_slice`, or `ctor` anywhere in the workspace
  3. All workspace dependencies use `default-features = false` and explicit feature lists; `cargo tree --duplicates` is empty (run in CI)
  4. Read-only inventory of upstream module names is documented in `.planning/upstream-inventory.md`: MCP code locations, channel registration mechanism, gateway REST/WS module split, telemetry call sites
**Plans**: TBD

Notes:
- Establishes the shared `osagent-amqp` crate scaffold (Pitfall 14: one TLS-bootstrap helper, no per-service re-implementation) even though AMQP runtime is M2.
- Cross-cutting Refinement #1 propagates here: MCP boundary will be **structural crate exclusion**, not Cargo features. This phase prepares the topology; Phase 1.3 extracts the crate.

### Phase 1.3: MCP Boundary & 4-Layer CI Gate
**Goal**: Make the load-bearing safety property (wizard cannot exfiltrate Vault secrets via MCP) provable in CI on every PR — the milestone-defining green check.
**Depends on**: Phase 1.2
**Requirements**: WS-02, WS-03
**Success Criteria** (what must be TRUE):
  1. `osagent-tools-mcp` exists as a dedicated crate; `bins/wizard/Cargo.toml` has zero `mcp` references (no dependency edge, no feature flag, no `cfg` gate, no optional dep); `bins/engineer/Cargo.toml` explicitly depends on `osagent-tools-mcp`
  2. 4-layer CI gate runs on every PR and release build, all four layers must pass; any single failure breaks the build:
      - L1: `grep -rnE '#\[cfg\(feature\s*=\s*"mcp"\)\]|use .*mcp' bins/wizard/ crates/wizard-*/` is empty
      - L2: `nm --defined-only target/release/osagent-wizard | grep -iE 'mcp|model[_-]?context[_-]?protocol|stdio_mcp|sse_mcp'` is empty
      - L3: `cargo bloat --crates --release --bin osagent-wizard` does not list any `mcp` crate
      - L4: `strings target/release/osagent-wizard | grep -iE 'mcp|stdio_mcp_server|sse_mcp'` is empty
  3. The 4-layer gate runs against BOTH isolated `cargo build -p osagent-wizard` AND workspace `cargo build --workspace` builds (catches feature-unification regressions)
  4. CI passes green on a hollow-wizard-at-M1 scaffold (~50 LOC `main.rs`) — the gate is binding even before wizard runtime fills in at M3
**Plans**: TBD

Notes:
- **This is the milestone-defining gate.** M1 cannot be marked complete until the 4-layer CI gate is green.
- Cross-cutting Refinement #2 propagates here: gate is 4-layer (not single `nm` grep) because LTO inlines, `strip = "symbols"` removes locals, non-deterministic DCE (rust #150462) makes single-layer non-binding.
- Establishing the gate BEFORE strips means subsequent strips cannot regress the property — they have to keep CI green.

### Phase 1.4: Whole-Crate Drops & Telemetry Audit
**Goal**: Shrink the attack surface fast via mechanical whole-crate deletion and audit/remove all upstream phone-home.
**Depends on**: Phase 1.3
**Requirements**: STRIP-01, STRIP-06, TELEMETRY-01
**Success Criteria** (what must be TRUE):
  1. `zeroclaw-hardware`, `robot-kit`, `aardvark-sys`, `apps/tauri`, `zeroclaw-plugins` are physically deleted from the workspace; `cargo build --workspace` still succeeds; binary size and compile time both measurably drop
  2. Webhook channel source is physically deleted (not feature-gated); cargo-deny includes a check preventing re-introduction without a documented decision
  3. `docs/telemetry-audit.md` documents every outbound HTTP call site found in the codebase (`reqwest::Client::new`, `sentry::`, `posthog::`, `honeycomb::`, env-driven URLs like `SENTRY_DSN`/`POSTHOG_KEY`); all identified phone-home paths are removed at source level
  4. `cargo deny` configuration extended to ban sentry, posthog, honeycomb, opentelemetry-exporter-otlp-http, and any reqwest-based telemetry crates as direct deps; `cargo tree -e normal` audit confirms no transitive telemetry survives; CI test under `unshare -n` asserts no startup network requirement
  5. 4-layer wizard-no-MCP gate (Phase 1.3) still passes
**Plans**: TBD

Notes:
- Parallel-safe with channel/provider/tool strips conceptually, but sequenced after Phase 1.3 so the MCP gate is binding before any code deletion.
- Pitfall 20 (telemetry survival in transitive deps) is the highest hidden risk here.

### Phase 1.5: Source Strips & MANIFEST Emission
**Goal**: Reduce channels/providers/tools to v1 surface and produce a build-time `MANIFEST.toml` that does not lie about what is compiled in.
**Depends on**: Phase 1.4
**Requirements**: STRIP-02, STRIP-03, STRIP-04, STRIP-07, MANIFEST-01, MANIFEST-02, MANIFEST-03
**Success Criteria** (what must be TRUE):
  1. 26 channel implementations physically deleted; 6 keepers remain in source (Telegram, Slack, Mattermost, Matrix, WhatsApp-Cloud, Signal); ~50 provider implementations deleted, 5 keepers remain (Anthropic, Gemini, Kimi-code via OpenAI-compatible base, Ollama, OpenRouter); ~35 tool implementations deleted per STRIP-04 list; non-en Fluent locale files deleted, en-US `.ftl` files retained as authoritative source
  2. Build emits `MANIFEST.toml` next to each binary with two sections: `[declared]` derived from `CARGO_FEATURE_*` env vars + Cargo.toml dependency tree at build time, and `[detected]` derived from post-link symbol analysis (`cargo bloat --crates`); CI fails on `[declared] != [detected]` divergence (catches "feature declared but code orphaned" and "code linked but not declared")
  3. `osagent manifest --diff <config.toml>` CLI subcommand exists on both binaries and refuses-to-start on bidirectional mismatch (config asks for capability binary lacks OR binary contains capability config did not authorize)
  4. Reproducible-build profile pinned: `[profile.release]` sets `codegen-units = 1`, `lto = "fat"`, `strip = "symbols"`, `panic = "abort"`; CI builds with `CARGO_INCREMENTAL=0`; two-run byte-equality assertion passes in the release job (mitigates rust #150462 non-deterministic DCE)
  5. 4-layer wizard-no-MCP gate (Phase 1.3) still passes
**Plans**: TBD

Notes:
- **Highest-risk phase.** PITFALLS.md's 24 named pitfalls concentrate here — Pitfall 8 (MANIFEST lies), Pitfall 17 (implicit features deprecation / `dep:` prefix), Pitfall 19 (non-deterministic DCE), Pitfall 20 (telemetry transitive survival).
- Memory backend strip (qdrant, postgres, embeddings, consolidation, community-skill HTTP) folded into the tool/provider strip pass.
- Mechanically simple given Pattern 1 (explicit registration) is already in place from Phase 1.2: deletion is `rm -rf` + removing `registry.register(...)` lines.
- **Closeout note (2026-06-17):** `build.rs` MANIFEST emission and binary `--manifest-diff` wiring are deferred to M2 Phase 2.1 (MANIFEST-04) and Phase 2.7 (MANIFEST-05). The osagent-manifest crate API + reproducibility profile shipped in 1.5.

### Phase 1.6: Gateway Fork & Install Drop-In
**Goal**: Fork the gateway crate to a minimal `/ws/chat`-only surface and open a drop-in ansible install task PR against sovereign-shield-install-guide.
**Depends on**: Phase 1.5
**Requirements**: STRIP-05, INSTALL-01
**Success Criteria** (what must be TRUE):
  1. `crates/osagent-gateway-ws-only/` exists as a forked-from-zeroclaw-gateway crate with REST endpoints (`/config`, `/onboarding`, `/pairing`, `/personality`, `/plugins`, `/webauthn`), ACP bridge, SSE, embedded web dashboard, pairing dashboard UI, mTLS server option, and outbound webhook endpoints all physically source-deleted; `/ws/chat` endpoint and `paired_tokens` auth path are kept and verifiably wire-compatible with OS-MDashboard's `chat-relay.ts`; old `zeroclaw-gateway` removed from workspace members
  2. PR opened against `sovereign-shield-install-guide` main containing `ansible/install_osagent.yml` as a structural template for the M2-completing engineer-binary install, NOT MERGED at M1 (merge happens at M2 close when engineer reaches parity)
  3. PR description contains a Touch/Change/Impact/Rollback plan posted BEFORE any ansible file is touched, per install-guide CLAUDE.md hard rule
  4. `install_osagent.yml` respects all 14 install-guide invariants: phase ordering preserved, mTLS cert provisioning pattern (sandbox-allowed home-mirror paths), `ExecStartPre` preflight, AMQP env-file pattern, pre-create audit log file before non-root daemon opens it, `StartLimitIntervalSec`/`StartLimitBurst` on systemd unit, `daemon-reload + restart` handler chain on env-file change, `get_url + checksum: sha256:` for binary install, clean-VM CI test against fresh `ubuntu:24.04` container
  5. 4-layer wizard-no-MCP gate (Phase 1.3) still passes; M1 ships
**Plans**: TBD

Notes:
- **ALL edits to sovereign-shield-install-guide require Touch/Change/Impact/Rollback plan in PR description before any file is touched**, per that repo's CLAUDE.md hard rule (plan-then-execute overrides auto-mode in install-guide work).
- Always cut a fresh branch off `master` of install-guide; stash unrelated dirty files first.
- Cross-UID ownership: bridge tool / install task must not chown into bind-mounted trees without explicit chown-back to outer-dir owner (Pitfall 12).
- PR is opened-not-merged because engineer binary is not yet at parity; merge is M2's exit gate.

### Phase 2.1: Native AMQP Bridge + MANIFEST build.rs emission
**Goal**: Stand up the engineer's native AMQP bridge tool (replacing the bash+python3+shell wrapper chain) and emit a build-time `MANIFEST.toml` that doesn't lie — the two foundations every subsequent M2 phase composes on.
**Depends on**: M1 (Phases 1.1–1.6 shipped)
**Requirements**: ENG-BRIDGE-01, ENG-BRIDGE-02, ENG-BRIDGE-03, ENG-BRIDGE-04, MANIFEST-04
**Success Criteria** (what must be TRUE):
  1. `crates/osagent-bridge/` workspace member exposes a `bridge_call` tool implemented on `lapin` 0.5 + manual `tokio-rustls` 0.27 stream with `ServerName` override to `rabbitmq.shield.internal` SAN while dialing `127.0.0.1`; engineer binary registers it, wizard binary does NOT depend on the crate (compile-time exclusion verified by `cargo tree -p osagent-wizard | grep osagent-bridge` empty)
  2. Operator allowlist at `/etc/zeroclaw/operator/allowlist.json` is loaded on engineer startup; missing file fails the daemon closed (exit non-zero) — verified by an integration test that boots with no allowlist and asserts non-zero exit + structured error log; every `bridge_call` invocation validates the verb against the 45-verb allowlist before publishing to RMQ
  3. Bash + python3 + bridge shell wrapper chain (the `shell_tool` + AMQP env-var path used by current zeroclaw engineer) is removed from the engineer's tool set; CI grep gate prevents re-introduction of `engineer-amqp-bridge` shell-out via a `tests/test-02.1-bridge.sh` assertion
  4. `build.rs` in `bins/engineer/` and `bins/wizard/` emits `target/release/MANIFEST-osagent-{engineer,wizard}.toml` with `[declared]` (from `CARGO_FEATURE_*` env + Cargo.toml direct deps tree) and `[detected]` (from `cargo bloat --crates --release --bin <binary>` JSON post-link analysis); CI gate `manifest-equality-check` asserts `[declared] == [detected]` per binary and fails the build on divergence
  5. Engineer `main.rs` gains a `clap`-based subcommand router (`osagent engineer <subcmd>`) — scaffolds the routing surface that Phase 2.7's `manifest --diff`, Phase 2.3's `audit verify`, and M4's `rotate-channel-secret` will plug into; routing skeleton has a `--help` integration test
**Plans**: TBD

Notes:
- **Foundation phase for M2.** Every later phase depends on this directly (2.5 exchange shares AMQP) or transitively (2.7 needs subcommand routing + emitted manifest; 2.8 parity audit covers all 45 verbs).
- **SAN override correctness is load-bearing.** rabbitmq runs on the platform's `shield.internal` cert with SAN `rabbitmq.shield.internal`; we dial `127.0.0.1` for path-locality, so the rustls `ServerName` must be overridden manually (Pitfall 14: one TLS-bootstrap helper in `osagent-amqp` scaffold from Phase 1.2 — reuse it here).
- **Fail-closed on missing allowlist is non-negotiable.** A permissive fallback would silently widen the engineer's effective verb set if the allowlist file is moved/renamed by an ops mistake.
- **MANIFEST-04 folded here** because (a) build.rs is independent of runtime feature code so it can land first, (b) the manifest equality gate gives every later phase the "did this strip orphan a feature?" signal for free, (c) MANIFEST-05 in Phase 2.7 needs MANIFEST-04 + subcommand routing both present.
- **PR #5 carry-over**: M2-PRE-RESEARCH.md is on `gsd/v0.2-engineer-init` via recovery chain; if PR #5 hasn't merged by the time 2.1 starts, branch from `gsd/v0.2-engineer-init` not `osagent-main`. Do NOT block 2.1 on PR #5 merge.

### Phase 2.2: SQLCipher Memory
**Goal**: Replace plaintext SQLite memory with SQLCipher-encrypted storage keyed by `customer_id` + Vault-supplied salt, so cross-customer restore fails fast.
**Depends on**: Phase 2.1 (engineer binary now has a real Cargo.toml dep graph to extend; allowlist + bridge wiring give us Vault access)
**Requirements**: ENG-SQLCIPHER-01, ENG-SQLCIPHER-02, ENG-SQLCIPHER-03, ENG-SQLCIPHER-04
**Success Criteria** (what must be TRUE):
  1. `crates/zeroclaw-memory/Cargo.toml` rusqlite dependency carries the `bundled-sqlcipher-vendored-openssl` feature (bundled = reproducible build, matches M1 Phase 1.5 reproducibility profile); existing `SqliteMemory` struct extended to call `PRAGMA key = ?` on open; build is green and `nm` on engineer binary shows linked SQLCipher symbols
  2. KDF (PBKDF2-SHA256 with documented iteration count, OR scrypt N=16384 r=8 p=1 — choice documented in `crates/zeroclaw-memory/src/sqlcipher.rs` rustdoc) derives a 32-byte key from `customer_id` + 32-byte Vault-fetched salt; salt is fetched at daemon start via `bridge_call` and held in memory only — never persisted, never logged
  3. Cross-customer restore integration test: two fixture DBs encrypted for `customer_a` + `customer_b` respectively; opening A with B's key returns `SQLITE_NOTADB` immediately (no partial read, no panic); test committed to `crates/zeroclaw-memory/tests/cross_customer_restore.rs`
  4. Ansible re-encrypt migration task: one-shot pre-`ExecStart` step in `install_osagent.yml` that exports any existing plaintext `engineer.db`, drops the file, re-creates encrypted-with-customer-key, re-imports; idempotent (no-op when already encrypted, detected by `PRAGMA cipher_version` returning non-empty); `tests/test-02.2-sqlcipher.sh` covers all 4 success criteria
  5. 4-layer wizard-no-MCP gate (Phase 1.3) still passes (verifies SQLCipher addition didn't pull in MCP-adjacent crates)
**Plans**: TBD

Notes:
- **Parallel-eligible with Phase 2.3.** Both touch persistent storage but independent code paths (memory vs audit log). Can run in alternating context blocks if needed; ROADMAP shows 2.2 first for numbering only.
- **Vault salt fetch path.** Engineer reads salt from `secret/data/<customer_id>/osagent/memory_salt` via the bridge tool, NOT via a direct Vault client — Vault writes are wizard's M3 responsibility; engineer only reads in M2.
- **Migration is one-shot.** Once a customer's memory is encrypted, the re-encrypt step is a no-op forever. The brittleness is "what if Vault is down at re-encrypt time?" — answer: ansible task gates on Vault reachability via existing install-guide preflight, refuses to proceed if unreachable.

### Phase 2.3: Audit Hash-Chain Dual-Sink
**Goal**: Make tampering with the engineer's audit log detectable end-to-end by writing every audit event to both journald and an append-only per-customer file, sha256-chaining each line to the previous, and anchoring the daily tail to the witness service.
**Depends on**: Phase 2.1 (subcommand routing scaffold needed for `audit verify` CLI; allowlist for witness-anchor verb)
**Requirements**: ENG-AUDIT-01, ENG-AUDIT-02, ENG-AUDIT-03, ENG-AUDIT-04
**Success Criteria** (what must be TRUE):
  1. Every audit-tagged event (tool dispatch, denied dispatch, Vault read, channel inbound/outbound, pause/resume, daemon start/stop) writes to BOTH `tracing-journald` (structured) AND append-only file at `/var/log/sovereign-shield/osagent-<customer_id>.audit` (mode 0640, `engineer:audit` ownership pre-created by ansible per install-guide pattern); integration test asserts dual-sink presence by tailing both journald + file for the same event
  2. Each file-sink line carries `prev_hash` field = `sha256(canonical_json(previous_event_payload))`; genesis line at daemon-start writes `prev_hash = "0".repeat(64)`; canonical-JSON serialization is deterministic (sorted keys, no whitespace variation) and committed as a tested helper in `crates/osagent-audit/src/canonical_json.rs`
  3. Daily cron unit (`/etc/systemd/system/osagent-witness-anchor.timer` shipped by install_osagent.yml) invokes `osagent engineer audit anchor` which submits the day's last `prev_hash` to ola-management-witness via `bridge_call witness.anchor.submit`; failure to anchor logs a critical event but does NOT halt the engineer daemon (witness outage must not take down engineer)
  4. `osagent engineer audit verify [--from <date>]` subcommand walks the audit file from the requested date forward, recomputes each line's expected `prev_hash`, reports the first mismatch with line number + event id; CI integration test seeds a corrupted chain (flip one byte in line N) and asserts `verify` exits non-zero pointing at line N
  5. 4-layer wizard-no-MCP gate (Phase 1.3) still passes
**Plans**: TBD

Notes:
- **Parallel-eligible with Phase 2.2.** New `crates/osagent-audit/` workspace member (or extension of existing zeroclaw-memory audit.rs — implementation decides per pre-research's `crates/zeroclaw-runtime/src/security/audit.rs` and `crates/zeroclaw-memory/src/audit.rs` existing surfaces).
- **Witness-anchor failure must be soft.** Treating witness unreachability as fatal would let a witness outage cascade into engineer outage — opposite of defense-in-depth. Critical event, retry on next daily run, surface in operator dashboard.
- **journald rotation lossiness is the reason for dual-sink** (per PROJECT.md key decision). The file is the durable scrape source; journald gives structured search + emergency-debug visibility.

### Phase 2.4: Lifecycle Gates
**Goal**: Make hot maintenance windows safe — daemon obeys a pause-marker file without restart, every tool observes a `CancellationToken`, Vault writes complete the current idempotency-keyed transaction before halting (no torn writes).
**Depends on**: Phase 2.2 (encrypted memory) + Phase 2.3 (audit log) — pause-mid-transaction tests need something stateful to pause AND must record pause/resume events to audit
**Requirements**: ENG-LIFECYCLE-01, ENG-LIFECYCLE-02, ENG-LIFECYCLE-03, ENG-LIFECYCLE-04
**Success Criteria** (what must be TRUE):
  1. Daemon polls `/var/lib/sovereign-shield/engineer.pause` and `/var/lib/sovereign-shield/engineer.active` marker files at 1s cadence; presence of `.pause` triggers `CancellationToken` flip and pause/resume events logged to audit; ansible install creates the directory with `engineer:engineer` 0750 ownership pre-`ExecStart` per install-guide pattern
  2. `CancellationToken` plumbed into every tool's `execute` signature via the `zeroclaw-runtime/src/agent/agent.rs` extension points already partially scaffolded in M1 (per M2-PRE-RESEARCH.md line 48); tool implementations check the token at each await point and `Err::Cancelled` cleanly rather than panicking; CI grep gate `no-blocking-execute-without-token` ensures every new tool added in M2/M3/M4 wires the token
  3. Vault-write tools that are mid-transaction at pause-time complete their current idempotency-keyed transaction (write + record idempotency key in encrypted memory) before observing the cancel — verified by integration test that (a) starts a Vault write, (b) drops `.pause` marker between the write's two phases, (c) asserts the write completes AND the next attempted write halts AND no torn idempotency key in memory
  4. Heartbeat (`zeroclaw-runtime/src/heartbeat/engine.rs`) and cron (`zeroclaw-runtime/src/cron/mod.rs`) job semantics respect the pause-marker: no new jobs spawned while paused; in-flight jobs observe the token; verified by `tests/test-02.4-lifecycle.sh` end-to-end
  5. 4-layer wizard-no-MCP gate (Phase 1.3) still passes
**Plans**: TBD

Notes:
- **The "Vault write complete current transaction then halt" semantic is the load-bearing safety property.** Halting mid-transaction would leave the idempotency key un-recorded in memory, so a resume would re-execute the Vault write and break the at-most-once guarantee. The integration test in success criterion 3 is non-negotiable.
- **CancellationToken infrastructure is ~60% scaffolded per pre-research** (`zeroclaw-runtime/src/agent/loop_.rs` already uses CancellationToken; `zeroclaw-runtime/src/cron/mod.rs` has a `pause_job` function). The new work is wiring it through every tool's `execute` and the Vault-write transaction coordination.
- **No daemon restart for maintenance windows is the whole point.** A restart-based pause loses in-flight conversation state in encrypted memory and tears any in-flight bridge call. The marker-file primitive is what makes hot maintenance safe.

### Phase 2.5: Exchange Channel
**Goal**: Replace HEARTBEAT.md's `.plan` / `.mission` / `.report` file-polling with a first-class typed AMQP `exchange` channel — durable per-customer queues, dead-letter routing for malformed envelopes.
**Depends on**: Phase 2.1 (shared AMQP infrastructure from `osagent-bridge` + `osagent-amqp`); Phase 2.3 (audit log records inbound/outbound envelopes)
**Requirements**: ENG-EXCHANGE-01, ENG-EXCHANGE-02, ENG-EXCHANGE-03
**Success Criteria** (what must be TRUE):
  1. `crates/zeroclaw-channels/src/exchange.rs` (or new `crates/osagent-exchange/` per implementation decision) implements an `ExchangeChannel` with three typed envelope schemas (`Plan`, `Mission`, `Report`) — serde-derived Rust structs serialized as AMQP messages with content-type `application/json` and explicit schema-version header; durable per-customer queues `exchange.<customer_id>.{plan,mission,report}.in` declared at channel start with `durable=true`, `auto-delete=false`
  2. Malformed envelope handling: messages that fail deserialization route to `exchange.<customer_id>.<verb>.dlq` with the original payload + error reason in headers; engineer logs a structured warning to audit but does NOT halt; DLQ messages re-driveable via `osagent engineer exchange replay --queue=<name>` (CLI may slip to M4 if scope tight — flag as out-of-scope on plan if so)
  3. HEARTBEAT.md file-polling code paths removed from `zeroclaw-runtime/src/heartbeat/engine.rs`; engineer no longer scans `/var/lib/zeroclaw/engineer/` for `.plan`/`.mission`/`.report` files; CI grep gate prevents re-introduction via `tests/test-02.5-exchange.sh`
  4. Channel registers in `zeroclaw-channels` orchestrator; engineer config example `examples/engineer-config.toml` shows `[channels.exchange]` block with bare-minimum keys (`enabled`, `customer_id`, `amqp_url`); engineer fails-to-start (Phase 2.7's manifest --diff gate) if config references a verb the binary's exchange channel doesn't carry
  5. 4-layer wizard-no-MCP gate (Phase 1.3) still passes (exchange channel does NOT pull in MCP)
**Plans**: TBD

Notes:
- **Parallel-eligible with Phase 2.6 after 2.1 lands.** Both depend on 2.1 but not on each other; can interleave across context blocks. Exchange channel is closer to the bridge tool semantically (shares AMQP infrastructure), so 2.5 numbering before 2.6.
- **DLQ replay CLI is nice-to-have.** If Phase 2.5 plan tracking shows scope creep, the `exchange replay` subcommand can defer to M4 as `OPS-EXCHANGE-REPLAY` without invalidating ENG-EXCHANGE-01..03 — flag explicitly in plan if you take this path.
- **Schema versioning is non-negotiable.** Future PLAN/MISSION/REPORT shape changes (likely in M3 when wizard sends PLANs) need backward-compatible deserialization; header-based versioning lets us evolve without breaking running deployments.

### Phase 2.6: Mattermost + Matrix Runtime + Channel Roles + Gateway Trim
**Goal**: Wire Mattermost (v4 REST + WS) and Matrix (matrix-sdk 0.18) runtime into engineer's channel orchestrator with `ops`/`observer` role enforcement at tool-dispatch, and opportunistically trim the gateway crate's dead sub-surface references (canvas::, sse::, acp::, static_files::, api_pairing::) while wiring engineer to the surviving `/ws/chat` + `paired_tokens` + node discovery surfaces.
**Depends on**: Phase 2.1 (engineer Cargo.toml dep graph + subcommand router); Phase 2.3 (audit log for denied dispatch events)
**Requirements**: ENG-CHANNELS-RT-01, ENG-CHANNELS-RT-02, ENG-CHANNELS-RT-03, ENG-CHANNEL-ROLES-01, ENG-CHANNEL-ROLES-02, ENG-CHANNEL-ROLES-03
**Success Criteria** (what must be TRUE):
  1. Mattermost channel runtime: in-house `reqwest` wrapper around Mattermost v4 REST API for outbound + WebSocket subscription for incoming events; auth via personal access token fetched from Vault via `bridge_call` at channel start; reconnect-on-WS-close with exponential backoff capped at 60s; integration test against a containerized mattermost instance OR mocked WS server asserts inbound message → ask-user-tool handle round-trip
  2. Matrix channel runtime: `matrix-sdk` bumped from 0.16 (M1 upstream) → 0.18; breaking-change matrix audited in plan against existing `crates/zeroclaw-channels/src/matrix.rs` (sync loop, E2EE rooms, encrypted upload, allowed_users); compile-green and integration test against a mocked homeserver asserts E2EE room receive + send round-trip; `cargo deny` advisory check passes on the new version
  3. Both runtimes integrate with the M1 `AskUserTool` channel-map handle (already wired in M1 to receive populated handles at orchestrator start) — engineer can prompt via Mattermost or Matrix and receive replies; both register in `zeroclaw-channels` orchestrator; engineer config example shows both `[channels.mattermost]` and `[channels.matrix]` blocks
  4. `ops` / `observer` role allowlist enforced at tool-dispatch time (NOT in the LLM prompt — defense in depth): config schema in `zeroclaw-channels` adds `[channels.<name>.roles]` map of `<channel_user_id> = "ops"|"observer"`; unauthorized attempts return a refusal message in-channel AND log a structured audit event with `caller_identity`, `attempted_tool`, `channel`, `reason="role_denied"`; default role for unmapped identities is `observer` (deny-by-default); integration tests cover both `ops`-permitted and `observer`-denied paths
  5. Gateway sub-surface coherent trim: dead references in `crates/zeroclaw-gateway/src/lib.rs` to `canvas::handle_*`, `sse::`, `acp::`, `static_files::`, `api_pairing::`, `api_webauthn::` removed; `CanvasStore` stub in `zeroclaw-runtime::tools` removed; `crates/zeroclaw-gateway/` now compiles cleanly with `cargo build -p zeroclaw-gateway`; engineer binary links the gateway's surviving `/ws/chat` + `paired_tokens` + node discovery surfaces (verified by `cargo tree -p osagent-engineer | grep zeroclaw-gateway` non-empty)
  6. 4-layer wizard-no-MCP gate (Phase 1.3) still passes (channel runtimes do NOT pull in MCP)
**Plans**: TBD

Notes:
- **Parallel-eligible with Phase 2.5 after 2.1 lands.**
- **Role enforcement at tool-dispatch (not LLM prompt) is non-negotiable.** Prompt-layer enforcement is bypassable via prompt injection or model jailbreak; tool-dispatch enforcement runs in Rust regardless of what the LLM was talked into asking for.
- **matrix-sdk 0.16 → 0.18 breaking changes**: olm encryption traits and event-handler signatures changed; plan must include the compile-error matrix and the per-error fix. M2-PRE-RESEARCH.md called out this bump as a known cost.
- **Gateway trim is opportunistic.** Per M1 closeout audit (v1-M1-MILESTONE-AUDIT.md "Intentionally deferred to M2: gateway sub-surface rewrite"), engineer wiring to the surviving gateway surface is the natural point to coherently trim dead references rather than salami-slicing on `osagent-main` in isolation. If scope tight, split into a 2.6.1 follow-up — but the `CanvasStore` stub removal is a hard checklist item for M2 close.
- **Mattermost WebSocket support**: current `crates/zeroclaw-channels/src/mattermost.rs` is REST polling per pre-research line 64. WS upgrade is part of this phase.

### Phase 2.7: MANIFEST `--diff` CLI Wiring
**Goal**: Wire `osagent engineer manifest --diff <config.toml>` subcommand so the engineer refuses to start when its compiled-in capability set diverges from what the config asks for — the user-facing teeth of MANIFEST-04's build-time gate.
**Depends on**: Phase 2.1 (subcommand router scaffold + MANIFEST emission from build.rs)
**Requirements**: MANIFEST-05
**Success Criteria** (what must be TRUE):
  1. `osagent engineer manifest --diff <config.toml>` subcommand wired in engineer's `main.rs` via the Phase 2.1 clap router; reads the binary-adjacent `MANIFEST-osagent-engineer.toml` (location resolved from `current_exe()` at runtime); calls `osagent_manifest::manifest_diff` (M1 Phase 1.5 API); reports added/missing/diverged entries
  2. Engineer daemon startup invokes `manifest_diff` against the loaded config BEFORE any channel/provider/tool initialization; refuse-to-start (exit code 1) on any mismatch (config asks for a channel/provider/tool the binary lacks OR binary contains a capability config did not authorize); structured error log names the specific entries on both sides
  3. Integration test: ship a config asking for a non-existent channel `mythical`; assert engineer exits 1 with a log line naming `mythical` as missing-from-binary. Reverse case: ship a binary that emits MANIFEST listing channel `slack` but config omits `slack`; assert engineer exits 1 with `slack` flagged as unauthorized-but-compiled-in
  4. CLI smoke test under `--manifest-diff path/to/engineer-config.toml` against the actually-built engineer binary in `target/release/` (CI step after `workspace build (engineer + wizard)`); shows zero-diff on the canonical install-guide config used by `install_osagent.yml`
  5. 4-layer wizard-no-MCP gate (Phase 1.3) still passes
**Plans**: TBD

Notes:
- **Scope-narrow phase by design.** MANIFEST-05 is a single REQ-ID; folding it into 2.1 would balloon 2.1's plan count and conflate "stand up bridge" with "wire user-facing CLI." Keep them separate so 2.1 can ship without waiting on 2.7's integration tests.
- **Wizard variant deferred to M3.** Per REQUIREMENTS.md MANIFEST-05: "Engineer binary only at M2 (wizard variant at M3)." Wizard's `osagent wizard manifest --diff` lands when wizard binary fills with runtime in M3.
- **The refuse-to-start path is the safety property.** Without runtime enforcement, MANIFEST-04's `[declared] == [detected]` build-time gate doesn't protect against config-drift attacks where an operator's config asks the binary to do something it can't honor.

### Phase 2.8: Engineer Parity Audit + Smoke Tests
**Goal**: Prove engineer can replace zeroclaw engineer in production — write a zero-⚠️ parity audit document, then validate on clean-VM and upgrade-in-place against the actually-shipped binary.
**Depends on**: Phases 2.1–2.7 (all engineer runtime features must land first — bridge, memory, audit, lifecycle, exchange, channels+roles, manifest CLI)
**Requirements**: ENG-PARITY-01, ENG-PARITY-02, ENG-PARITY-03
**Success Criteria** (what must be TRUE):
  1. `.planning/M2-PARITY-AUDIT.md` enumerates every current-zeroclaw-engineer capability: 45-verb allowlist coverage (one row per verb), HEARTBEAT.md behaviors (file-polling now removed in 2.5, plan/mission/report semantics now via exchange), scheduled-job semantics from `ola-host-engineer-config`, life-config integration points (skills catalog, RAG, scheduled jobs); each row carries status ✅ implemented in osagent-engineer / ⚠️ gap (with remediation plan) / ❌ not in scope (with reason citing M3/M4 deferral or PROJECT.md Out-of-Scope); zero ⚠️ before Phase 2.9 fires
  2. Clean-VM smoke test (`tests/e2e/clean-vm-smoke.sh` + fixture under `tests/e2e/fixtures/clean-vm-config/`): deploy `osagent-engineer` to a fresh Ubuntu 22.04 VM via `install_osagent.yml`, run all 45 bridge verbs once, verify journald + audit file entries for each, verify SQLCipher memory persists across `systemctl restart engineer`, verify exchange channel can receive a test PLAN envelope; test committed and runnable from CI under a containerized Ubuntu 22.04 image
  3. Upgrade-in-place smoke test (`tests/e2e/upgrade-vm-smoke.sh`): same fixture, but starting from a VM already running zeroclaw engineer with non-empty plaintext memory; deploy osAgent-engineer via `install_osagent.yml`; verify ansible re-encrypt migration (Phase 2.2 ENG-SQLCIPHER-04) preserves all memory rows; verify zeroclaw engineer service stopped (`systemctl is-active zeroclaw` == `inactive`) and binary removed from `/usr/local/bin/`; verify osAgent engineer service is active and serving on the same systemd unit name
  4. M2-PARITY-AUDIT.md status section at top reads `Result: ✅ All 45 verbs implemented; 0 ⚠️; ready for MIG-ENG cutover.` — this is the human-readable green light for Phase 2.9
  5. 4-layer wizard-no-MCP gate (Phase 1.3) still passes
**Plans**: TBD

Notes:
- **The zero-⚠️ gate is non-negotiable per REQUIREMENTS.md ENG-PARITY-01.** Any ⚠️ in the parity audit blocks Phase 2.9; remediation either adds a follow-up plan in 2.8 or explicitly defers the gap to M4 with a documented reason.
- **Smoke tests run the actually-shipped binary**, not a debug build. CI must `cargo build --release` and use that artifact for both VM tests; pre-research's `bins/engineer/Cargo.toml` placeholder is by then filled with all M2 deps so the release build is real.
- **Upgrade-in-place is where SQLCipher migration earns its rent.** If 2.2's re-encrypt step is brittle, 2.8 catches it before customers do.
- **Parity audit is a `/gsd:plan` candidate, not a free-form doc.** Plan it as TDD — define the audit shape (one section per verb, schema for each row), then fill rows.

### Phase 2.9: Engineer Migration Cutover
**Goal**: Ship the engineer to production — set the binary URL + SHA256 in `install_osagent.yml`, merge `feat/osagent-install-task` PR, swap the playbook reference from `install_zeroclaw.yml` → `install_osagent.yml`, and purge the old zeroclaw engineer binary on next ansible apply.
**Depends on**: Phase 2.8 (zero-⚠️ parity + green smoke tests are the hard prerequisite per REQUIREMENTS.md ENG-PARITY-01)
**Requirements**: MIG-ENG-01, MIG-ENG-02
**Success Criteria** (what must be TRUE):
  1. `osagent-engineer` v0.2.0 release published (self-hosted runner per PROJECT.md decision #40; signed; binary URL + SHA256 known and pinned); `sovereign-shield-install-guide/ansible/install_osagent.yml` updated on `feat/osagent-install-task` branch with the real binary URL + SHA256 replacing the M1 placeholder; `meta:end_play` guard removed from `install_osagent.yml`; PR description carries a fresh Touch/Change/Impact/Rollback plan per install-guide CLAUDE.md hard rule
  2. PR `feat/osagent-install-task` on `sovereign-shield-install-guide` merges to master via the install-guide repo's standard review process; merge commit lands; CI green on the install-guide repo's own clean-VM ansible-lint + idempotency tests
  3. Install-guide playbook reference for the engineer task swapped from `install_zeroclaw.yml` → `install_osagent.yml`: the old `import_tasks: install_zeroclaw.yml` line is **deleted** (not commented out — per REQUIREMENTS.md MIG-ENG-02); CI grep gate in install-guide repo confirms zero remaining references to `install_zeroclaw.yml` for the engineer role
  4. Old zeroclaw engineer binary symlink `/usr/local/bin/zeroclaw` purged on next `ansible-playbook site.yml` run on a target VM: ansible task `name: Purge legacy zeroclaw engineer binary` removes both `/usr/local/bin/zeroclaw` and the systemd unit `/etc/systemd/system/zeroclaw.service` (after stopping); idempotent (no-op if already absent); verified on the Phase 2.8 upgrade-in-place smoke test fixture
  5. STATE.md updated: M2 closed; engineer in production on at least one customer's deployment; PROJECT.md "Validated" section gains an M2 entry; ROADMAP.md milestone row flips to ✅
**Plans**: TBD

Notes:
- **Plan-then-execute discipline applies harder here than anywhere else.** Per `sovereign-shield-install-guide/CLAUDE.md`, every install-guide edit needs Touch/Change/Impact/Rollback in the PR description BEFORE any file is touched. This is the user's hard rule and overrides auto-mode per the memory note `feedback_plan_then_execute_overrides_auto_mode.md`.
- **Always cut a fresh branch off master of install-guide.** Per memory note `feedback_install_guide_always_new_branch.md`. The existing `feat/osagent-install-task` branch from M1 is the exception — that's the PR being merged, not a new branch.
- **Sharp cutover, not coexist.** Per PROJECT.md decision #30 and key-decisions table: "Production migration via sharp cutover; coexist doubles maintenance forever." The old reference deletion (not commenting) in MIG-ENG-02 is the structural enforcement of this.
- **PR #141 (install-guide) holding from M1**: this phase is where the M1 PR's `meta:end_play` guard finally comes off and the PR ships.
- **No referral-chain wording**: per memory note `feedback_no_written_referral_chain_when_poaching.md` — irrelevant here (no vendor-swap pitch in this PR), noted for completeness.

## Progress

**Execution Order (M1):**
Phases executed in numeric order: 1.1 → 1.2 → 1.3 → 1.4 → 1.5 → 1.6 (all shipped 2026-06-17)

**Execution Order (M2):**
Phases execute in numeric order with parallel-eligibility flagged:
- 2.1 (foundation) → 2.2 (memory) ∥ 2.3 (audit) → 2.4 (lifecycle, needs 2.2+2.3)
- 2.1 → 2.5 (exchange) ∥ 2.6 (channels + roles + gateway trim)
- 2.1 → 2.7 (manifest CLI, needs 2.1's main.rs scaffold)
- 2.1 → … → 2.7 → 2.8 (parity audit + smoke tests, needs everything)
- 2.8 → 2.9 (migration cutover)

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1.1 Fork & Attribution & Sync Runbook | 4/4 | ✅ Complete | 2026-06-17 |
| 1.2 Workspace Skeleton & Binary Split | n/n | ✅ Complete | 2026-06-17 |
| 1.3 MCP Boundary & 4-Layer CI Gate | n/n | ✅ Complete | 2026-06-17 |
| 1.4 Whole-Crate Drops & Telemetry Audit | n/n | ✅ Complete | 2026-06-17 |
| 1.5 Source Strips & MANIFEST Emission | n/n | 🟡 Partial (MANIFEST emission deferred to 2.1) | 2026-06-17 |
| 1.6 Gateway Fork & Install Drop-In | n/n | ✅ Complete | 2026-06-17 |
| 2.1 Native AMQP Bridge + MANIFEST build.rs | 0/TBD | Not started | - |
| 2.2 SQLCipher Memory | 0/TBD | Not started | - |
| 2.3 Audit Hash-Chain Dual-Sink | 0/TBD | Not started | - |
| 2.4 Lifecycle Gates | 0/TBD | Not started | - |
| 2.5 Exchange Channel | 0/TBD | Not started | - |
| 2.6 Mattermost+Matrix Runtime + Roles + Gateway Trim | 0/TBD | Not started | - |
| 2.7 MANIFEST `--diff` CLI Wiring | 0/TBD | Not started | - |
| 2.8 Engineer Parity Audit + Smoke Tests | 0/TBD | Not started | - |
| 2.9 Engineer Migration Cutover | 0/TBD | Not started | - |

---

*Roadmap created: 2026-06-12*
*Granularity: standard (6 phases for M1, within 5-8 band; 9 phases for M2, within 6-9 expected band)*
*M1 coverage: 20/20 v1 requirements mapped, 0 orphans*
*M2 coverage: 32/32 v1 requirements mapped, 0 orphans*
*Phase 1.1 planned: 2026-06-12 — 4 plans, 2 waves (wave 0 preflight + wave 1 fork + wave 2 deny.toml/CI || UPSTREAM_SYNC.md)*
*M2 roadmap appended: 2026-06-17 (phases 2.1–2.9; sequencing per M2-PRE-RESEARCH.md constraints #1–#9; MANIFEST-04 folded into 2.1, gateway sub-surface coherent trim folded into 2.6 per M1 closeout audit recommendation)*
