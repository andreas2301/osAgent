# Requirements: osAgent

**Defined:** 2026-06-12 (M1)
**Updated:** 2026-06-17 (M2 v0.2-engineer initialization)
**Core Value:** Wizard cannot exfiltrate Vault secrets via an MCP server because the wizard binary has compile-time zero MCP, verified by a 4-layer CI gate (source-grep + `nm --defined-only` + `cargo-bloat --crates` + `strings`).

This document scopes **M2 — v0.2-engineer** as the current milestone (v1 block). M3/M4 requirements remain in the v3/v4 blocks below (promoted when those milestones begin via `/gsd:new-milestone`).

---

## v1 Requirements (M2 — Engineer binary production-ready)

### Native AMQP bridge

- [ ] **ENG-BRIDGE-01**: A standalone `osagent-bridge` crate (added to workspace members) exposes a native `bridge_call` tool. Implementation uses `lapin` 0.5 + manual `tokio-rustls` 0.27 stream for `ServerName` override (`rabbitmq.shield.internal` SAN, dial `127.0.0.1`).
- [ ] **ENG-BRIDGE-02**: Operator allowlist (`/etc/zeroclaw/operator/allowlist.json`, 45-verb) is loaded at engineer startup; missing file fails the daemon closed (no fallback to permissive). All bridge calls validate verb against allowlist before publishing.
- [ ] **ENG-BRIDGE-03**: `bridge_call` registered on the engineer binary tool registry; wizard binary does NOT register it (compile-time exclusion via crate membership).
- [ ] **ENG-BRIDGE-04**: Bash + python3 + bridge shell wrapper chain (current zeroclaw approach via `shell_tool` + AMQP env vars) is removed from the engineer's tool set. Native tool is the only path.

### Exchange channel

- [ ] **ENG-EXCHANGE-01**: First-class `exchange` channel implementation: PLAN, MISSION, REPORT envelope schemas (typed Rust structs serialized as AMQP messages), durable per-customer RMQ queues, retry + dead-letter routing for malformed envelopes.
- [ ] **ENG-EXCHANGE-02**: Replaces the file-polling pattern in HEARTBEAT.md (engineer no longer scans for `.plan` / `.mission` / `.report` files; receives them via exchange channel subscriptions).
- [ ] **ENG-EXCHANGE-03**: Channel registers in `zeroclaw-channels` orchestrator; engineer config enables it via `[channels.exchange]`.

### Lifecycle gates

- [ ] **ENG-LIFECYCLE-01**: Daemon-level pause-marker file (`/var/lib/sovereign-shield/engineer.pause`) and activation-marker file (`/var/lib/sovereign-shield/engineer.active`) coordinate hot maintenance windows without daemon restart.
- [ ] **ENG-LIFECYCLE-02**: `CancellationToken` plumbed into every tool's `execute` method via the existing `zeroclaw-runtime/src/agent/agent.rs` extension points. Pause sets the token; tools observe and halt cleanly.
- [ ] **ENG-LIFECYCLE-03**: Vault-write tools that are mid-transaction at pause-time complete their current idempotency-keyed transaction (avoiding torn writes), then halt before starting any new write. Verified by integration test.
- [ ] **ENG-LIFECYCLE-04**: Heartbeat + cron job semantics respect the pause-marker (no new jobs spawned while paused).

### SQLCipher memory

- [ ] **ENG-SQLCIPHER-01**: `zeroclaw-memory` SQLite backend rebuilt on `rusqlite` with `bundled-sqlcipher-vendored-openssl` feature (bundled = reproducible build).
- [ ] **ENG-SQLCIPHER-02**: Encryption key derived from `customer_id + Vault-supplied salt` (PBKDF2 or scrypt; choice documented in implementation). Salt fetched from Vault at daemon start; key never persisted to disk.
- [ ] **ENG-SQLCIPHER-03**: Cross-customer restore fails fast — opening a DB encrypted for customer A with customer B's key returns `SQLITE_NOTADB` immediately (verified by integration test with two fixture DBs).
- [ ] **ENG-SQLCIPHER-04**: Migration path from existing plaintext SQLite memory: ansible task includes a one-shot re-encrypt step on first install (export, drop, re-create encrypted, re-import).

### Audit hash-chain

- [ ] **ENG-AUDIT-01**: Audit log dual-sink: every audit-tagged event writes to both `journald` (via `tracing-journald`) AND append-only file at `/var/log/sovereign-shield/osagent-<customer_id>.audit` (mode 0640, owned `engineer:audit`).
- [ ] **ENG-AUDIT-02**: Each file-sink line carries a `prev_hash` field linking it to the previous line's hash (`sha256` of canonical-JSON event payload). Genesis line at daemon-start writes `prev_hash = "0" * 64`.
- [ ] **ENG-AUDIT-03**: Daily cron triggers `osagent-witness-anchor` which submits the day's last `prev_hash` to ola-management-witness (anchor cross-link). Failure to anchor logs a critical event but does NOT halt the daemon.
- [ ] **ENG-AUDIT-04**: Hash-chain verification tool `osagent engineer audit verify [--from <date>]` walks the chain and reports tampering. CI integration test seeds a corrupted chain to confirm detection.

### Mattermost + Matrix runtime

- [ ] **ENG-CHANNELS-RT-01**: Mattermost channel runtime — in-house `reqwest` wrapper around Mattermost v4 REST API + WebSocket for incoming events. Auth via personal access token (Vault-stored, rotated via M4's `OPS-ROTATE` later).
- [ ] **ENG-CHANNELS-RT-02**: Matrix channel runtime upgraded from upstream `matrix-sdk` 0.16 → 0.18 (version bump; verify breaking-change matrix against M1's existing matrix.rs in `zeroclaw-channels`). E2E-encrypted rooms supported.
- [ ] **ENG-CHANNELS-RT-03**: Both channels integrate with the M1 `AskUserTool` channel-map handle (already wired to receive populated handles at orchestrator start).

### Channel roles

- [ ] **ENG-CHANNEL-ROLES-01**: Per-channel role allowlist: `ops` (can trigger tool calls + decisions) and `observer` (read-only — receives notifications, cannot trigger). Config schema additions to `zeroclaw-channels`.
- [ ] **ENG-CHANNEL-ROLES-02**: Role enforced at tool-dispatch time (NOT at LLM-prompt time — defense in depth). Unauthorized attempt returns a refusal message AND logs the attempt to audit log with caller identity + attempted tool name.
- [ ] **ENG-CHANNEL-ROLES-03**: Default role for unspecified identities is `observer` (deny-by-default).

### Build-time MANIFEST emission (deferred from M1 Phase 1.5)

- [ ] **MANIFEST-04**: `build.rs` in each binary crate (`bins/engineer`, `bins/wizard`) emits `MANIFEST.toml` to `target/release/MANIFEST-<binary>.toml`. `[declared]` section derived from `CARGO_FEATURE_*` env vars + Cargo.toml direct deps. `[detected]` section derived from `cargo bloat --crates --release --bin <binary>` JSON output. CI gate enforces `[declared] == [detected]` per binary.
- [ ] **MANIFEST-05**: Binary subcommand `osagent {engineer,wizard} manifest --diff <config.toml>` validates that every channel/provider/tool referenced in config exists in the binary's `MANIFEST.toml`. Refuse-to-start (exit 1) on mismatch. Wired into engineer's `main.rs` at M2; wizard's at M3.

### Engineer parity + migration

- [ ] **ENG-PARITY-01**: Functional-parity audit document `.planning/M2-PARITY-AUDIT.md` enumerates: every current-zeroclaw-engineer capability (45-verb allowlist coverage, HEARTBEAT.md behaviors, scheduled-job semantics, life-config integration points from `ola-host-engineer-config`); for each, status = ✅ implemented in osagent-engineer / ⚠️ gap / ❌ not in scope (with reason). Zero ⚠️ before MIG-ENG can fire.
- [ ] **ENG-PARITY-02**: End-to-end smoke test: deploy `osagent-engineer` to a clean Ubuntu 22.04 VM via `install_osagent.yml`, run all 45 bridge verbs once, verify journald + audit file entries, verify SQLCipher memory persists across restart. Test fixture committed to `tests/e2e/`.
- [ ] **ENG-PARITY-03**: Upgrade-in-place smoke test: same as ENG-PARITY-02 but starting from a VM already running zeroclaw engineer. Verify migration succeeds without data loss; verify zeroclaw service stopped + binary removed from `/usr/local/bin/`.
- [ ] **MIG-ENG-01**: `sovereign-shield-install-guide/ansible/install_osagent.yml` PR (`feat/osagent-install-task`) updated with binary URL + SHA256 once `osagent-engineer` v0.2 release is published; `meta:end_play` guard removed; PR ships and merges.
- [ ] **MIG-ENG-02**: install-guide engineer task swapped from `install_zeroclaw.yml` reference to `install_osagent.yml` reference in the playbook; old reference deleted (not commented). Old zeroclaw engineer binary symlink `/usr/local/bin/zeroclaw` purged.

---

## v3 Requirements (M3 — Wizard binary + subagent system)

Deferred to M3.

### Wizard binary
- **WIZ-BIN**: Wizard binary builds with `osagent-tools-mcp` NOT in its dependency tree. CI 4-layer gate passes on every build.
- **WIZ-VAULT**: Vault tool with idempotency keys (hash of tool + args + correlation_id), customer-prefix path enforcement (`secret/data/<customer_id>/...`), structured approval-required wrapper. Uses `vaultrs` 0.8 + KV v2 + AppRole.
- **WIZ-2P**: 2-person approval primitive in Rust runtime: dashboard ack + chat ack from distinct identities; 1h timeout escalates to sysadmin chat (no auto-approve, no silent expiry).
- **WIZ-BOOT**: Bootstrap secret: sealed plaintext on disk mode 0600 root:wizard, loaded only when Vault unreachable at startup. Documented in `documentation/osAgent/BOOTSTRAP.md`.
- **WIZ-CHANNELS**: Dashboard WS (already works) + Telegram + Slack + customer-chosen one of {Mattermost, Matrix, WhatsApp-Cloud, Signal} active for wizard binary.

### Subagent primitive
- **SUB-FORMAT**: Markdown frontmatter format (Claude-Code convention). Parsed via `gray_matter` 0.3.
- **SUB-POOL**: Pool cost semantics — parent's daily cap is the shared pool. NOT per-subagent.
- **SUB-DEPTH**: One level deep enforced at primitive level (no grand-subagents).
- **SUB-AUDIT**: Both parent + subagent identity in audit log (`engineer/secret-rotation-planner`).
- **SUB-SIGN**: Subagent prompts signed by wizard's git commit signature (ssh-key 0.6 `SshSig` parses `git -c gpg.format=ssh` format); engineer verifies before invoke.
- **SUB-ISOLATE**: Subagent runs in separate Tokio task with own `CancellationToken`. No grand-children. Cost pool drains from parent's daily cap.

## v4 Requirements (M4 — Channels + Ops + Provider routing + Production rollout)

Deferred to M4.

### Remaining channels
- **CHAN-WA**: WhatsApp-Cloud in-house wrapper (no maintained Rust crate; in-house 200-LOC `reqwest` wrapper around Meta Graph API).
- **CHAN-SIGNAL**: Signal channel runs `signal-cli` (GPLv3 Java daemon) as a separate process. JSON-RPC over Unix socket. License boundary is the process edge (mere-aggregation, not derived work). Java/JVM ansible provisioning ships with install task. AGPL Rust SDKs explicitly banned in `cargo deny`.
- **CHAN-OUTBOX**: SQLite per-channel outbox; replay on reconnect. Survives "Telegram country-block + dashboard down" combo.
- **CHAN-ROTATE**: `osagent rotate-channel-secret --channel=<name>` CLI rotates bot tokens in Vault + restarts process + updates external app config.

### Codeword challenge
- **CHAL-01**: High-risk tool calls emit 4-word phrase shown in dashboard, require confirm-reply in channel. Configurable risk threshold per tool.

### Provider routing
- **PROV-MODES**: Three provider policy modes — `cloud-first` (default), `local-first` (oracle primary, cloud fallback), `local-only` (hard wall — refuse to serve if oracle unreachable, emit alert, NEVER silent failover). Type-level separation enforced (LocalOnlyProvider vs FallbackProvider) so `local-only` cannot accidentally call cloud.
- **PROV-ORACLE**: `[providers.models.oracle]` config entry for ola-management-oracle (Ollama-compatible local LLM proxy).

### Operational CLIs
- **OPS-RESCUE**: `osagent-rescue` CLI: same operator allowlist, no LLM, direct AMQP. Saves you when daemon is crash-looping at 3am.
- **OPS-ROTATE**: `osagent rotate-channel-secret` CLI (see CHAN-ROTATE).
- **OPS-MANIFEST**: `osagent manifest --diff` CLI extended to wizard binary case (M2 ships engineer-only via MANIFEST-05).

### Documentation & production rollout
- **DOC-FULL**: `sovereign-shield-backup/documentation/osAgent/` complete: architecture, bootstrap, approval flow, audit format, subagent spec, rotation runbook, rescue runbook, upstream sync runbook, channel onboarding per customer. Arc42 numbering convention.
- **MIG-WIZ**: install-guide ansible task adds `install_osagent.yml` wizard binary deployment alongside engineer. Both running in production. Old zeroclaw binaries removed from PATH and `/usr/local/bin/zeroclaw` symlink purged.

---

## Out of Scope

(Unchanged from M1 — all M1 boundaries remain valid M2 boundaries.)

| Feature | Reason |
|---------|--------|
| **Microsoft Teams channel** | Not in zeroclaw v0.7.5; requires Bot Framework + Azure AD app (2–3 weeks net-new). Defer to post-v4. |
| **APAC corporate channels** (Lark/Feishu, WeCom, DingTalk, WeChat, QQ, LINE) | No customer demand; revisit when APAC GTM begins. |
| **Webhook ingress channel** | User rejection on security grounds: "rather build a custom osAgent update for a weird stack than overengineer and give the capabilities to get hacked." Custom integration per weird-stack customer instead. |
| **WhatsApp Web (Selenium scraper)** | Brittle, TOS-violating. WhatsApp-Cloud only. |
| **In-process Rust Signal SDK** | All available Rust Signal crates (presage, libsignal-service, libsignal) are AGPL-3.0; including any forces entire osAgent under AGPL. signal-cli subprocess is the only license-compatible path. |
| **Custom Landlock sandbox re-enable** | v1 keeps `sandbox.enabled=false` matching current install-guide. Auto-detect chain explicitly disabled. v2 candidate. |
| **Subagent grand-children (depth > 1)** | Fork-bomb hazard; one-level enforced. |
| **Outbound webhook subscriptions** | Same security posture as inbound webhook channel. |
| **Public artifact distribution** (signed binaries on GitHub Releases) | Ship via our own infrastructure first; v2. |
| **Multi-tenancy inside one osAgent** | Design assumes one customer = one platform = one engineer + one wizard. Multi-tenancy at config layer rejected at config-load. |
| **MCP on wizard binary** | Compile-time-prohibited (structural crate exclusion + 4-layer CI gate). Load-bearing safety property. |
| **Coexist forever** (old zeroclaw + osAgent both supported) | Explicit sharp cutover; coexist doubles maintenance forever. |
| **Browser tool, web_search, web_fetch, image gen, voice (Call+Wake), PDF RAG, WebAuthn, Postgres/Qdrant memory, embeddings consolidation, community skill HTTP fetch** | None used by current install-guide; zero customer demand. |
| **Cargo `inventory!` / `linkme!` distributed-slice registration** | Auto-discovery at link time defeats the structural MCP exclusion. Explicit registration only. |
| **AGPL-licensed dependencies (transitively or directly)** | License contamination would force entire osAgent under AGPL. `cargo deny` enforces license allowlist. |

---

## Validated Requirements (M1 — Foundation, shipped 2026-06-17)

All M1 v1 requirements validated by shipping. See `.planning/v1-M1-MILESTONE-AUDIT.md` and PROJECT.md's Validated section for the full closeout.

| Requirement | Phase | Status |
|-------------|-------|--------|
| FORK-01 | 1.1 | ✅ Complete |
| FORK-02 | 1.1 | ✅ Complete (PR awaiting merge) |
| FORK-03 | 1.1 | ✅ Complete |
| WS-01 | 1.2 | ✅ Complete |
| WS-02 | 1.3 | ✅ Complete (MILESTONE-DEFINING) |
| WS-03 | 1.3 | ✅ Complete (4-layer gate green) |
| WS-04 | 1.2 | ✅ Complete |
| WS-05 | 1.2 | ✅ Complete |
| STRIP-01 | 1.4 | ✅ Complete |
| STRIP-02 | 1.5 | ✅ Complete |
| STRIP-03 | 1.5 | ✅ Complete |
| STRIP-04 | 1.5 | ✅ Complete |
| STRIP-05 | 1.6 | ✅ Complete |
| STRIP-06 | 1.4 | ✅ Complete |
| STRIP-07 | 1.5 | ✅ Complete |
| TELEMETRY-01 | 1.4 | ✅ Complete |
| MANIFEST-01 | 1.5 | ⚠️ Scaffold complete; build.rs emission rolled into M2 MANIFEST-04 |
| MANIFEST-02 | 1.5 | ⚠️ Crate API complete; binary wiring rolled into M2 MANIFEST-05 |
| MANIFEST-03 | 1.5 | ✅ Complete (reproducibility profile pinned) |
| INSTALL-01 | 1.6 | ✅ Complete (PR awaiting M2 binary URL + SHA256) |

---

## Traceability (M2)

Empty initially. Filled by roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| ENG-BRIDGE-01..04 | TBD | Pending |
| ENG-EXCHANGE-01..03 | TBD | Pending |
| ENG-LIFECYCLE-01..04 | TBD | Pending |
| ENG-SQLCIPHER-01..04 | TBD | Pending |
| ENG-AUDIT-01..04 | TBD | Pending |
| ENG-CHANNELS-RT-01..03 | TBD | Pending |
| ENG-CHANNEL-ROLES-01..03 | TBD | Pending |
| MANIFEST-04 | TBD | Pending |
| MANIFEST-05 | TBD | Pending |
| ENG-PARITY-01..03 | TBD | Pending |
| MIG-ENG-01..02 | TBD | Pending |

**Coverage:**
- v1 (M2) requirements: 32 atomic (across 11 requirement groups)
- Mapped to phases: 0 (pending roadmap creation)
- Unmapped: 32 ⚠️ (will be 0 after roadmap)

---
*Requirements defined: 2026-06-12 (M1)*
*Last updated: 2026-06-17 (M2 v0.2-engineer initialization, M1 closeout)*
