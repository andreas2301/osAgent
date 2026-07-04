# osAgent M2/M3/M4 Design Spec

**Date:** 2026-07-02  
**Repo:** `andreas2301/osAgent` (`osagent-main`)  
**Scope:** Turn the M1 structural scaffold into a full replacement for the current `zeroclaw`-based engineer and wizard agents on the Sovereign Shield appliance.  
**Prerequisite:** M1 must be cargo-green (`cargo build --workspace` passes on `osagent-main`) before M2 work begins.

---

## 1. Context

- M1 produced two compile-time-separated binaries: `osagent-engineer` and `osagent-wizard`.
- The wizard binary provably contains zero MCP code (4-layer CI gate: source-grep, `nm`, `cargo-bloat`, `strings`).
- M2/M3/M4 are currently expressed as intent in `.planning/REQUIREMENTS.md`. This document turns that intent into implementable specs with acceptance criteria.
- Current production still runs upstream `zeroclaw` v0.7.5 via `install_zeroclaw.yml`. Cutover is sharp; we do not support coexistence forever.

## 2. Design principles

1. **Fail loud.** Every trust-boundary violation, config mismatch, or missing credential refuses to start with a clear log line.
2. **No silent fallbacks.** Provider routing modes are type-level; `local-only` cannot accidentally call a cloud provider.
3. **Compile-time guarantees beat runtime checks.** Wizard MCP exclusion stays structural, not feature-flagged.
4. **Operator is the sole privileged executor.** The engineer runtime asks; the Operator service (privileged, sudoers) allows or denies.
5. **One envelope schema.** All agent↔platform internal messaging uses `PLAN` / `MISSION` / `REPORT` envelopes over the exchange channel.

## 3. M2 — Engineer binary production-ready

### 3.1 Goal
Replace the current `zeroclaw`-based engineer with `osagent-engineer` without regressing any capability used by the install guide.

### 3.2 Components

#### 3.2.1 Native AMQP bridge (`ENG-BRIDGE`)
- Engineer binary links `lapin` and opens its own mTLS connection to RabbitMQ.
- Uses manual `tokio-rustls` stream so the dial host can be `127.0.0.1` while the TLS ServerName is `rabbitmq.shield.internal` (matches SAN).
- Reads AMQP credentials from the same `EnvironmentFile` path used today: `/etc/zeroclaw/engineer-amqp.env`.
- Publishes operator requests to the existing operator queue; the Operator service keeps the allowlist and executes or rejects commands.
- On startup, fails closed if it cannot establish the AMQP connection or the operator queue is unreachable.

#### 3.2.2 Exchange channel (`ENG-EXCHANGE`)
- Internal AMQP channel carrying typed envelopes.
- Envelope kinds:
  - `PLAN` — proposed work item, references a plan file or inline payload.
  - `MISSION` — customer request, references a mission file or inline payload.
  - `OPERATION` — privileged command request sent by engineer to operator.
  - `REPORT` — status/audit output, references a report file or inline payload.
- Each envelope has: `correlation_id`, `sender` (agent identity), `recipient`, `timestamp_utc`, `payload_schema_version`, `payload`.
- Engineer uses the exchange channel natively for:
  - operator command requests/responses,
  - heartbeat reports,
  - mission status updates.
- The wizard→strategist file-drop bridge converts MISSION/PLAN file drops into exchange envelopes before publishing to the Strategist queue.

#### 3.2.3 Lifecycle gates (`ENG-LIFECYCLE`)
- `/etc/sovereign-shield/agent-pause` and `/etc/sovereign-shield/heartbeat-enabled` are watched via inotify or polled every 5s.
- A global `CancellationToken` is passed into every async tool.
- On pause: complete the in-flight Vault/memory transaction, then halt; resume on marker removal.
- On activation: validate config and certs before resuming work.

#### 3.2.4 sqlcipher memory (`ENG-SQLCIPHER`)
- Memory backend uses `rusqlite` with `bundled-sqlcipher-vendored-openssl`.
- Database path: `/home/engineer/.zeroclaw/memory/osagent-engineer.db`.
- Key derived via HKDF-SHA256 from `customer_id` + Vault-supplied salt fetched at startup.
- If customer_id or salt is missing, the daemon refuses to start (`fail loud`).
- Cross-customer restore must fail fast: store a `customer_fingerprint` hash in the DB header and compare on open.

#### 3.2.5 Hash-chained audit log (`ENG-AUDIT`)
- Dual sink:
  1. journald via `tracing-journald` or structured stdout captured by systemd.
  2. Append-only file: `/var/log/sovereign-shield/osagent-<customer_id>.audit`.
- Each entry: `seq_no`, `timestamp_utc`, `event_type`, `actor`, `correlation_id`, `payload_hash`, `prev_hash`.
- File pre-created by ansible with `0640 root:engineer` so the non-root daemon can append.
- Daily anchor hash cross-linked to the witness service's chain.

#### 3.2.6 Channel runtimes (`ENG-CHANNELS-RT`)
- Mattermost: in-house `reqwest` wrapper around v4 REST + WebSocket.
- Matrix: `matrix-sdk` 0.18.
- Telegram and dashboard WebSocket keep the existing implementations.

#### 3.2.7 Channel roles (`ENG-CHANNEL-ROLES`)
- Per-channel allowlist of `ops` (can trigger actions) and `observer` (read-only) identities.
- Identity is chat handle/user-id, not osAgent internal user.
- Default: all configured allowed users are `observer`; `ops` list must be explicitly populated.

### 3.3 Config additions

```toml
[bridge]
amqp_url = "amqps://127.0.0.1:5671/operator"
ca_file = "/home/engineer/.zeroclaw/certs/engineer-amqp/ca.crt"
cert_file = "/home/engineer/.zeroclaw/certs/engineer-amqp/tls.crt"
key_file = "/home/engineer/.zeroclaw/certs/engineer-amqp/tls.key"
tls_server_name = "rabbitmq.shield.internal"
operator_queue = "operator.requests"
allowlist_path = "/etc/zeroclaw/operator/allowlist.json"

[memory]
backend = "sqlcipher"
vault_salt_path = "secret/data/<customer_id>/osagent/engineer-memory-salt"

[audit]
file_path = "/var/log/sovereign-shield/osagent-<customer_id>.audit"
daily_anchor_hour = 0

[channels.matrix]
homeserver_url = "..."
access_token = "..."
room_id = "..."
roles_ops = ["@admin:example.com"]
roles_observer = ["@user:example.com"]

[channels.mattermost]
server_url = "..."
token = "..."
team = "..."
channel = "..."
roles_ops = ["admin"]
roles_observer = ["user"]
```

### 3.4 Data flow: privileged command

1. User sends command to engineer via Telegram/dashboard.
2. Engineer decides the command requires privilege.
3. Engineer builds an `OPERATION` envelope with `recipient = "operator"` and the shell command as payload.
4. Engineer publishes envelope to the operator queue over AMQP.
5. Operator service receives envelope, verifies sender identity and allowlist.
6. Operator executes command under sudo, captures stdout/stderr/rc.
7. Operator publishes a `REPORT` envelope back to the engineer queue.
8. Engineer renders the result to the user.

### 3.5 Acceptance criteria

- [ ] `cargo build --workspace -p osagent-engineer` passes.
- [ ] `osagent-engineer` starts and connects to RabbitMQ over mTLS using the existing cert env files.
- [ ] Sending a privileged command from engineer reaches the Operator queue and is gated by `/etc/zeroclaw/operator/allowlist.json`.
- [ ] Heartbeat reports are emitted as `REPORT` envelopes when `/etc/sovereign-shield/heartbeat-enabled` exists.
- [ ] Pause marker halts new work within 5s and completes in-flight transactions.
- [ ] sqlcipher DB rejects a key derived from a different `customer_id`.
- [ ] Audit log file appends entries with monotonic `seq_no` and verifiable `prev_hash`.
- [ ] Mattermost and Matrix channels can receive a message and reply.
- [ ] Functional parity audit passes: for every capability the current zeroclaw engineer uses in the install guide, `osagent-engineer` has an equivalent.
- [ ] `install_osagent.yml` deploys `osagent-engineer` instead of `zeroclaw` for the engineer user.

---

## 4. M3 — Wizard binary + subagent system

### 4.1 Goal
Make `osagent-wizard` a secure customer-intake and approval orchestrator with no MCP dependency and a subagent primitive for delegated work.

### 4.2 Components

#### 4.2.1 Vault tool (`WIZ-VAULT`)
- Tool name: `vault_read`, `vault_write`, `vault_list`.
- Idempotency key = SHA256 of `tool_name + args + correlation_id`.
- Customer-prefix enforcement: all paths must start with `secret/data/<customer_id>/` or `secret/metadata/<customer_id>/`.
- Uses `vaultrs` 0.8 with KV v2 and AppRole auth.
- Structured `approval_required` wrapper: returns a pending-approval envelope if the path matches a high-risk pattern.

#### 4.2.2 2-person approval primitive (`WIZ-2P`)
- Any high-risk tool call emits a `PLAN` envelope requiring two distinct acks.
- Ack channels: dashboard button + chat message from an `ops`-role identity.
- 1-hour timeout; on expiry, escalate to sysadmin chat channel; no auto-approve.
- Both acks recorded in audit log with correlation_id.

#### 4.2.3 Bootstrap secret (`WIZ-BOOT`)
- Fallback path: `/etc/sovereign-shield/wizard-bootstrap-secret.json`.
- File mode `0600`, owner `root:wizard`.
- Loaded only when Vault is unreachable at startup.
- Contents: encrypted customer secrets; key derived from a host-bound TPM/LUKS secret if available, otherwise from a machine-id derived key.
- If bootstrap file is world-readable or group-readable beyond `wizard`, the daemon refuses to start.

#### 4.2.4 Subagent primitive (`SUB-*`)
- Subagent definition file: Markdown with YAML frontmatter (Claude-Code convention).
- Frontmatter fields: `name`, `purpose`, `parent`, `max_cost_usd`, `allowed_tools`, `allowed_repos`, `signature`.
- Pool semantics: subagent cost drains from the parent's daily cap.
- Depth enforcement: one level only; subagent cannot spawn another subagent.
- Audit: both parent and subagent identity logged on every action.
- Signing: wizard signs the prompt with `git -c gpg.format=ssh commit -S`; engineer verifies the `SshSig` with `ssh-key` 0.6 before invoking.
- Isolation: subagent runs in a separate Tokio task with its own `CancellationToken`.

#### 4.2.5 Wizard channels (`WIZ-CHANNELS`)
- Dashboard WebSocket (`/ws/chat`) kept from gateway.
- Telegram.
- Slack.
- Mattermost.
- Matrix.
- WhatsApp-Cloud.
- Signal.
- All channels can be configured; one is the primary inbound channel, the rest are fallback-ordered.

### 4.3 Config additions

```toml
[vault]
addr = "https://127.0.0.1:8200"
approle_role_id = "..."
approle_secret_id = "..."
customer_id = "..."

[approval]
high_risk_path_patterns = ["secret/data/<customer_id>/production/*"]
ack_timeout_seconds = 3600
escalation_channel = "sysadmin-alerts"

[bootstrap]
path = "/etc/sovereign-shield/wizard-bootstrap-secret.json"

[subagent]
max_depth = 1
daily_cost_cap_usd = 100.0

[channels]
primary = "telegram"
fallbacks = ["slack", "matrix", "dashboard"]
```

### 4.4 Data flow: customer request → mission

1. Customer messages wizard on Telegram.
2. Wizard parses intent and decides a mission is needed.
3. Wizard builds a `MISSION` envelope.
4. If the mission requires Vault access or high-risk action, the 2-person approval gate fires.
5. After both acks, wizard writes the MISSION file to `/opt/sovereign-shield/exchange/wizard-strategist/`.
6. `wizard-strategist-bridge` converts the file to a `MISSION` exchange envelope and publishes to Strategist.
7. Strategist processes mission and writes a `REPORT` reply.
8. Bridge converts REPORT to a file in `/opt/sovereign-shield/exchange/strategist-to-wizard/`.
9. Wizard reads the reply and responds to the customer.

### 4.5 Acceptance criteria

- [ ] `cargo build --workspace -p osagent-wizard` passes.
- [ ] 4-layer wizard-no-MCP gate still passes.
- [ ] Wizard can read a Vault secret under `secret/data/<customer_id>/...`.
- [ ] Wizard rejects a Vault path outside the customer prefix.
- [ ] High-risk Vault path triggers 2-person approval; no execution without two distinct acks.
- [ ] Approval timeout escalates to sysadmin channel; no silent expiry.
- [ ] Bootstrap secret loads only when Vault is unreachable.
- [ ] Subagent prompt with invalid signature is rejected.
- [ ] Subagent cannot spawn a grand-subagent.
- [ ] Subagent cost is deducted from parent daily cap.
- [ ] Wizard can send and receive messages on Telegram, Slack, Mattermost, Matrix, WhatsApp-Cloud, and Signal.

---

## 5. M4 — Channels + ops + provider routing + production rollout

### 5.1 Goal
Complete the channel surface, add operational CLIs, implement provider routing policies, and cut over the wizard.

### 5.2 Components

#### 5.2.1 WhatsApp-Cloud (`CHAN-WA`)
- In-house `reqwest` wrapper around Meta Graph API.
- Webhook verification and message sending.
- Phone number ID and access token from Vault.

#### 5.2.2 Signal (`CHAN-SIGNAL`)
- `signal-cli` Java daemon run as a separate process.
- osAgent communicates via JSON-RPC over Unix socket.
- JVM provisioned by ansible.
- License boundary: mere aggregation; no AGPL Rust SDK.

#### 5.2.3 Channel outbox (`CHAN-OUTBOX`)
- SQLite per-channel outbox at `/home/wizard/.zeroclaw/outbox/<channel>.db`.
- Messages queued on disconnect; replay on reconnect.
- Survives Telegram country-block + dashboard down combo.

#### 5.2.4 Channel secret rotation (`CHAN-ROTATE`)
- CLI: `osagent rotate-channel-secret --channel=<name>`.
- Reads new secret from Vault, writes to channel config, restarts channel task.

#### 5.2.5 Codeword challenge (`CHAL-01`)
- High-risk tool calls emit a 4-word phrase shown in dashboard.
- User must reply with the phrase in the chat channel.
- Configurable risk threshold per tool.

#### 5.2.6 Provider routing (`PROV-*`)
- Three type-level policies:
  - `cloud-first`: current behavior (Gemini → Kimi → Anthropic → Ollama fallback).
  - `local-first`: Oracle/Ollama primary, cloud fallback.
  - `local-only`: Oracle/Ollama only; hard fail with alert if unreachable.
- Config entry: `[providers.models.oracle]` for `ola-management-oracle` (Ollama-compatible proxy).
- Type separation: `LocalOnlyProvider` vs `FallbackProvider` so `local-only` cannot accidentally call cloud.

#### 5.2.7 Rescue CLI (`OPS-RESCUE`)
- Binary: `osagent-rescue`.
- No LLM; direct AMQP to operator.
- Same operator allowlist as engineer.
- For 3am recovery when daemon is crash-looping.

### 5.3 Acceptance criteria

- [ ] WhatsApp-Cloud channel can send and receive messages.
- [ ] Signal channel can send and receive messages via `signal-cli`.
- [ ] Outbox replays queued messages after reconnect.
- [ ] `osagent rotate-channel-secret` rotates a channel secret and restarts the channel without full daemon restart.
- [ ] Codeword challenge blocks high-risk tool calls until confirmed.
- [ ] `cloud-first`, `local-first`, and `local-only` provider modes behave correctly and are type-enforced.
- [ ] `osagent-rescue` can send one allowlisted operator command without loading the full agent runtime.
- [ ] `sovereign-shield-backup/documentation/osAgent/` contains architecture, bootstrap, approval flow, audit format, subagent spec, rotation runbook, rescue runbook, upstream sync runbook, and channel onboarding docs.
- [ ] `install_osagent.yml` deploys both engineer and wizard binaries; `zeroclaw` binary is removed from `/usr/local/bin`.

---

## 6. Security and trust boundaries

| Boundary | Enforcement |
|---|---|
| Wizard no MCP | 4-layer CI gate on `osagent-wizard` (compile-time). |
| Privileged execution | Operator service only; allowlist checked in Operator, not agent. |
| Vault path isolation | Customer prefix enforced by wizard tool. |
| Subagent depth | Runtime check; one level only. |
| Local-only provider | Type-level; `LocalOnlyProvider` has no cloud provider methods. |
| Audit immutability | Append-only file + journald; prev_hash chain. |
| Cert sandbox | Agent reads home-mirror certs; Operator reads `/opt/sovereign-shield/certs`. |

## 7. Testing strategy

- **Unit tests:** every new crate (bridge, exchange, memory, audit, vault tool, approval, subagent).
- **Integration tests:** engineer↔operator round-trip; wizard↔strategist file-drop round-trip; 2-person approval timeout; subagent signing.
- **Smoke tests:** existing `binary_smoke.rs` extended to verify `osagent-engineer` and `osagent-wizard` still identify correctly.
- **CI gates:** keep all 7 existing M1 gates; add M2/M3/M4-specific gates.
- **Live acceptance:** clean VM install via `install_osagent.yml`; upgrade-in-place from zeroclaw.

## 8. Rollout / cutover

1. **M2 close:** switch engineer only to `osagent-engineer`; keep wizard on zeroclaw.
2. **M3 close:** switch wizard to `osagent-wizard`; both binaries run in production.
3. **M4 close:** add remaining channels and provider modes; remove `/usr/local/bin/zeroclaw` and the old systemd units.
4. Each cutover has a rollback playbook: stop osagent unit, restore zeroclaw binary, restart zeroclaw unit.

## 9. Assumptions and open questions

- M1 cargo cleanup lands before M2 starts.
- Operator service already speaks AMQP and can accept `MISSION`/`REPORT` envelopes with minimal changes.
- `ola-management-oracle` local LLM proxy exists or will be built before M4.
- JVM installation for `signal-cli` is acceptable on the appliance.
- `customer_id` is available at daemon startup via config or Vault.

---

*Next step: review this spec. Once approved, invoke the `writing-plans` skill to break each milestone into implementation plans.*
