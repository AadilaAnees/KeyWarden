# AGENTS.md: KeyWarden build specification

> Drop this file in the repository root. It works as `AGENTS.md` (Codex, Copilot agent, many IDEs). For other tools, copy it to `CLAUDE.md` (Claude Code), `.cursor/rules/keywarden.mdc` (Cursor) or `.github/copilot-instructions.md`.

**Contents**
0. How to work (read first)
1. Project context and goals
2. Hard rules
3. Architecture and design decisions
4. Repository layout
5. Tech stack
6. Infrastructure: OpenBao, PostgreSQL, TLS, Docker Compose
7. Data model
8. Gateway API
9. ML service
10. Response engine and rotation
11. Canary (decoy) system
12. Scanner and migration engine
13. CLI and client libraries
14. Dashboard
15. DevOps and platform integration
16. Testing and acceptance
17. Documentation deliverables
18. Demo script
19. Milestones and build order
20. Coding standards
21. Things to verify against current docs
22. Glossary

---

## 0. How to work (read first)

You are building **KeyWarden**, a proof-of-concept secure credential management platform, for a student competition. Read this whole file before writing code.

1. **Work milestone by milestone** (section 19). Finish and test one before starting the next. Do not wait for approval between milestones. If a decision is not covered here, pick the documented default, record it in `docs/decisions.md` (one short entry: context, choice, reason) and continue. Stop and ask only if you are blocked.
2. **Priorities:** P0 = must work for the demo. P1 = strong differentiators, build after P0. P2 = nice to have. Never start a P1/P2 item while a P0 item is broken.
3. **Verify external facts before using them.** Tool versions, CLI flags, image names and API paths change. Section 21 lists known risk areas. When unsure, read the official documentation of the pinned version and adapt. Do not guess.
4. **Never fabricate results.** Every number in the docs (accuracy, latency, memory) must come from a run you actually did. Write the command and date next to it. The ML data is synthetic and the docs must say so.
5. **Keep it runnable.** After every milestone, the documented start command must bring the stack up from a clean checkout, and the tests must pass. `make` is only a convenience: it is not installed by default on Windows, so every Makefile target must also be documented as a plain `docker compose ...` / test command in `docs/setup.md`. Never make `make` a requirement.
6. **Small commits**, conventional commit messages (`feat:`, `fix:`, `docs:`, `test:`, `chore:`).
7. At the end of each milestone, append a short report to `docs/progress.md`: done, deferred, known issues.
8. The primary user is a student presenting this to a mentor and judges. Prefer clarity, a reliable demo and honest documentation over cleverness.

---

## 1. Project context and goals

### 1.1 Competition brief
- **Event:** TransitionX 2026, Dialog Axiata industry problem statement.
- **Problem:** *Password hardcoding in scripts.* Operational and automation scripts contain hardcoded passwords, API keys and service account credentials, which creates risks of credential leakage, unauthorized access and compliance violations. Commercial vaults may be too costly or hard to deploy.
- **Required:** an **open-source-based** secure credential management platform that:
  1. eliminates hardcoded credentials from scripts and automation tools;
  2. stores secrets centrally and securely;
  3. supports authentication and authorization controls;
  4. allows retrieval through APIs or command-line interfaces;
  5. supports credential rotation and expiration policies;
  6. provides audit trails for secret access and usage;
  7. integrates with Linux, Windows, cloud and DevOps environments.
- **Expected outcome:** a working proof of concept demonstrating secure secret storage, retrieval, rotation and auditing, using open-source technologies, with minimal operational complexity and cost.
- **Mentor guidance:** consider SSH keys; HSMs are expensive; certificate-based authentication; EPMS (enterprise password management); a secure storage module; key rotation. Deliverables: PoC, working test environment, open-source software, documentation explaining how everything is set up, including MFA, password complexity and credential theft.

### 1.2 Our solution (one paragraph)
KeyWarden covers the whole credential lifecycle on an open-source core (OpenBao). It **discovers** hardcoded secrets, **migrates** them into the vault (with auto-generated code patches), **protects** them (short-lived certificate identities, short-lived and single-use credentials), **detects** abuse (ML risk scoring plus decoy credentials) and **responds** automatically (alert, hold, rotate, quarantine), with a tamper-evident audit trail and a dashboard.

| Stage | What it does | Priority |
|---|---|---|
| Discover | Scan repos/script folders for hardcoded secrets (Gitleaks engine + custom rules) | P1 |
| Migrate | Import into vault; generate code patches; auto-rotate old values | P1 |
| Protect | Certificate identities, scoped single-use tokens, dynamic DB credentials, rotation/expiry | P0 |
| Detect | Isolation Forest + rules risk scoring with explanations; canary credentials | P0 |
| Respond | Graduated response engine, rotation hooks, quarantine | P0 |
| Observe | Audit trail (tamper-evident), React dashboard, attack simulator | P0 (basic) / P1 |

### 1.3 The five innovations (what judges should remember)
1. **Migration engine:** solves the adoption problem, not only storage.
2. **Canary credentials:** decoys that give near-zero false-positive alerts and traceable leak locations.
3. **Behavioural risk scoring with explanations and graduated response.**
4. **Secretless by default:** dynamic, short-lived, single-use credentials.
5. **Low cost:** runs on one small VM; no HSM; documented TPM/HSM upgrade path.

### 1.4 Non-goals
Not production-ready; no HA clustering; no HSM/TPM implementation (document only); no custom cryptography (use OpenBao and standard libraries only); no multi-tenant SaaS features.

### 1.5 Demo success criteria
A 5-minute live demo can show: scan and patch a legacy repo; a script fetching its secret via CLI with no secret in code; normal access (no alert); an abnormal access flagged with a plain-language reason and automatic rotation/quarantine; a canary hit giving an instant critical alert; the audit trail and rotation history on the dashboard; a CI pipeline fetching a secret without any stored token (P1).

---

## 2. Hard rules

### 2.1 Licensing (open source only)
- Every dependency must have an **OSI-approved** licence (MIT, Apache-2.0, BSD, MPL-2.0, PostgreSQL, ISC, etc.). Check before adding and record it in `docs/licenses.md` (name, version, licence, role).
- **Do not use** anything under BUSL/BSL, SSPL, RSAL, Elastic License or similar source-available licences. In particular: **not HashiCorp Vault** (use OpenBao), **not MongoDB** (use PostgreSQL), no Redis versions under non-OSI licences (avoid Redis entirely).
- The project itself is licensed **Apache-2.0** (`LICENSE` file).

### 2.2 Security hygiene (this is a security product, so dogfood it)
- No secrets, keys or tokens in the repository, ever. Use `.env.example` with placeholders.
- **OpenBao unseal keys and the root token are held by the human, outside the project folder (never in the workspace).** AI agents must never ask for, read, print, log or store them. Scripts that need a token must prompt the human or read it from the human's environment at run time. Service credentials generated for the stack (for example AppRole role IDs and secret IDs) may live in git-ignored `./.secrets/`, and must never be printed.
- Install a **gitleaks pre-commit hook** and a CI job on this repository itself.
- Never log secret values, tokens, private keys or full client certificates. Use a redaction layer for logs (keys: `value`, `secret`, `password`, `token`, `authorization`, `private_key`).
- Never include risk scores, reasons or policy details in responses to scripts (do not help an attacker tune an attack). Return generic errors to the data plane; details go to the admin API and dashboard.
- Validate all input (Zod/class-validator; Pydantic in Python). Parameterised queries only. Set request size limits, rate limits and timeouts everywhere.
- TLS for all inter-service traffic. Containers run as non-root, with minimal base images and pinned versions.
- Decoy credentials must use a custom prefix (`kwc_`). **Do not** generate strings in real provider formats (for example real AWS key formats).
- Use constant-time comparison for tokens/hashes. Store only hashes of enrollment tokens.

### 2.3 Honesty rules
- Document trade-offs and limitations in `docs/limitations.md`, for example: the gateway is a high-value component; rotation of generic KV secrets needs a hook to update the target system; the ML model is evaluated on synthetic data; slow-and-low attacks are harder to detect.
- Do not claim compliance with any standard. You may say the design "supports" practices.

---

## 3. Architecture and design decisions

### 3.1 Diagram

```
                    ┌──────────────────────────── control plane (8444, TLS, admin JWT/OIDC) ─────┐
 Admin ──browser──► │  Dashboard (React)  ◄──SSE──  Gateway admin API                              │
                    └──────────────────────────────────────┬──────────────────────────────────────┘
                                                           │
 Script / CI / Windows task                                │
   │  keywarden CLI / PS module                            ▼
   │   mTLS (short-lived client cert)       ┌──────────────────────────┐   score    ┌──────────────┐
   └────────────────────────────────────►  │   Gateway (NestJS)        │ ─────────► │ ML service   │
              data plane (8443)             │  authN · authZ · decide   │ ◄───────── │ FastAPI +    │
                                            │  telemetry · response     │  risk+why  │ scikit-learn │
                                            └───┬──────────┬───────────┘            └──────┬───────┘
                       scoped, 1-use tokens    │          │ events/alerts/actions          │ read-only history
                                               ▼          ▼                                ▼
                                       ┌────────────┐  ┌──────────────────────────────────────┐
                                       │  OpenBao   │  │ PostgreSQL (platform)                 │
                                       │ KV2 · PKI  │  └──────────────────────────────────────┘
                                       │ DB engine  │
                                       │ audit log  │──► audit tailer (gateway) ──► bypass detector
                                       └─────┬──────┘
                                             ▼
                                   PostgreSQL (demo target, dynamic creds)

   Decoy API / SSH honeypot ──canary hit (HMAC webhook)──► Gateway ──► alerts + response
```

### 3.2 Decisions (copy into `docs/decisions.md` as ADRs)
- **D1. OpenBao, not Vault.** Vault is BUSL-licensed (not OSI open source). OpenBao is the Linux Foundation, MPL-2.0 fork, API-compatible.
- **D2. Gateway as policy and telemetry broker.** Scripts never talk to OpenBao directly. The gateway authenticates the caller (mTLS cert or CI JWT), asks the ML service for a risk score, applies the response policy, then mints a **per-request, identity-scoped, single-use OpenBao token** (via a token role) and reads the secret with it. OpenBao still enforces per-identity policy and records identity metadata in its audit log. *Trade-off:* the gateway is a high-value target; document it and keep its own OpenBao policy minimal.
- **D3. Certificate identities with CSR enrollment.** Scripts generate their own key pair locally (ECDSA P-256); the private key never leaves the host. The gateway signs the CSR through OpenBao's PKI engine. Certificate lifetime is short (default 1 hour, configurable); renewal is automatic through `keywarden agent` or `keywarden renew`.
- **D4. Dual telemetry.** The gateway emits a structured event for every request (primary ML input). Separately, the OpenBao audit log is tailed to cross-check and to detect **bypass** (access to secrets not made through the gateway).
- **D5. Hybrid detection.** Rules (explainable, work from day 1) plus Isolation Forest on identity-relative features. Risk = max(rule risk, model risk). Every score carries human-readable reasons.
- **D6. Graduated response** (section 10), not a binary block.
- **D7. PostgreSQL** for platform data (replaces MongoDB, which is SSPL). Two instances: `pg-platform` and `pg-demo` (the demo target for dynamic credentials).
- **D8. Fail modes.** If the ML service is unreachable or slow (>150 ms), the gateway falls back to rule-only scoring and raises a "degraded detection" alert. `ML_FAIL_MODE=open` for low/medium sensitivity secrets, `closed` for high sensitivity.
- **D9. Rotation honesty.** Rotating a stored value is only meaningful if the target system is updated. Rotation goes through pluggable **hooks**; secrets without a hook are flagged "manual rotation required".
- **D10. Two listeners.** Data plane (8443, mTLS) and control plane (8444, admin) are separate processes/ports to reduce attack surface.

### 3.3 Threat model (summary; expand in `docs/threat-model.md`)
| Threat | Defence |
|---|---|
| Script/repo leaked | No secret inside; canary credentials alert on use; scanner finds and removes old ones |
| Client cert + key stolen from a host | Short lifetime; revocable; source-IP/time/frequency anomalies raise risk; quarantine |
| Insider bulk-pulls secrets | Frequency/new-secret features; per-identity least privilege; audit |
| Probing for secrets | Canary secret paths in vault and gateway; every denied attempt logged |
| Direct access to OpenBao bypassing gateway | Network isolation; bypass detector on audit log |
| Gateway compromise | Minimal gateway policy; short token TTLs; audit cross-check; documented residual risk |
| Audit log tampering | Hash-chained events (P1); OpenBao audit file as second source |
| Slow-and-low or mimicry attack | Known limitation; per-identity baselines, max-age caps; report honestly |
| Baseline poisoning ("boiling frog") | Retrain only on low-risk, non-alerted events; cap baseline window |

---

## 4. Repository layout

```
keywarden/
├── AGENTS.md  README.md  LICENSE  CHANGELOG.md  Makefile  .env.example  .gitignore
├── .gitleaks.toml  .pre-commit-config.yaml
├── docker-compose.yml  docker-compose.dev.yml
├── infra/
│   ├── openbao/        config.hcl  policies/*.hcl  bootstrap.sh  password-policy.hcl
│   ├── tls/            make-dev-ca.sh
│   ├── postgres/       platform-init.sql  demo-target-init.sql
│   ├── keycloak/       realm-export.json            (P2)
│   └── gcp/            vm-setup.md  firewall.md
├── services/
│   ├── gateway/        NestJS app (data plane + control plane)
│   ├── ml/             FastAPI app, training, features, eval
│   └── decoy/          decoy API (+ SSH honeypot P2)
├── cli/                keywarden CLI (TypeScript)
├── clients/
│   ├── python/         keywarden-py (tiny lib)
│   └── powershell/     KeyWarden.psm1 + tests
├── scanner/            Python package: scan, classify, import, patch
├── dashboard/          React + Vite + TS
├── demo/
│   ├── legacy-repo/    sample scripts with hardcoded secrets (py, sh, ps1, js, .env, yaml)
│   ├── fixed-repo/     expected "after" state (used by golden tests)
│   ├── attack-sim/     scenario scripts used by the dashboard simulator and e2e tests
│   └── scripts/        demo walkthrough helpers
├── ml-data/            synthetic generator, saved datasets (git-ignored), figures
├── docs/               (section 17)
├── tests/              e2e + security tests
└── .github/workflows/  ci.yml  demo-oidc.yml
```

---

## 5. Tech stack

Pin exact versions in lockfiles and Dockerfiles. At build time, use the **latest stable** release of each and record it in `docs/licenses.md`.

| Area | Choice | Notes |
|---|---|---|
| Secrets core | **OpenBao** (MPL-2.0), image `openbao/openbao`, pinned to **2.6.2** (the version whose Docker + Raft setup was verified); upgrade deliberately later and re-test | KV v2, PKI, Database engine, AppRole auth, declarative audit device, token roles, response wrapping. **Storage: Raft** (the `file` backend is deprecated and removed in 2.7.0; never use it) |
| Gateway | **NestJS + TypeScript** (Node LTS) | Prisma (PostgreSQL), `jose` (JWT), `@nestjs/schedule`, `zod`, `pino` logging, `helmet`, rate limiting |
| ML | **Python 3.12, FastAPI, scikit-learn, pandas, numpy, SQLAlchemy (read-only), matplotlib** | Isolation Forest, robust z-scores |
| Platform DB | **PostgreSQL** | Events, alerts, identities, rotations, etc. |
| Demo target DB | PostgreSQL (second instance) | Target of dynamic credentials |
| Scanner | Python, **Gitleaks** binary (MIT) + custom rules; `libcst` for Python rewrites | |
| CLI | TypeScript (Node), `commander`, `undici`/native fetch | Optional single binary later (P2) |
| PowerShell module | PowerShell 7+ and Windows PowerShell 5.1 compatible | |
| Dashboard | React 18, Vite, TypeScript, Tailwind CSS, shadcn/ui, Recharts, TanStack Query, React Router | SSE for live updates |
| Admin MFA | Keycloak (Apache-2.0) OIDC with TOTP | P2 (P0 uses dev auth mode) |
| Decoys | FastAPI decoy API; `asyncssh` honeypot (P2) | |
| Containers | Docker + Docker Compose | Compose profiles: `core`, `auth`, `demo` |
| CI | GitHub Actions | lint, test, gitleaks, license check |

---

## 6. Infrastructure

### 6.1 Docker Compose services

| Service | Image/build | Ports (host) | Network | Notes |
|---|---|---|---|---|
| `openbao` | `openbao/openbao:2.6.2`, command `server` (without it the image starts an in-memory dev server) | `127.0.0.1:8200` only | internal | Raft storage on a **named volume mounted at `/openbao/file`** (other paths fail with permission denied because the image runs as an unprivileged user); audit log on a **named volume at `/openbao/logs`**; config mounted read-only at `/openbao/config`; env `BAO_ADDR=http://127.0.0.1:8200`; healthcheck on `/v1/sys/health` (shows unhealthy while sealed, which is expected) |
| `pg-platform` | postgres | none | internal | Platform data; roles: `kw_app` (rw), `kw_ml` (read-only) |
| `pg-demo` | postgres | 5433 (localhost) | internal | Target for dynamic creds |
| `gateway` | build | 8443 (data), 8444 (admin) | internal + edge | Non-root; mounts audit log volume read-only |
| `ml` | build | none | internal | Not exposed |
| `decoy` | build | 8081 (API), 2222 (SSH, P2) | edge | Deliberately isolated from internal network |
| `dashboard` | build (nginx) | 8080 | edge | Talks to gateway admin API |
| `keycloak` | quay/keycloak | 8085 | edge | Profile `auth` (P2) |

Networks: `kw-internal` (OpenBao, DBs, ML, gateway) and `kw-edge` (gateway, dashboard, decoy). Add healthchecks and `depends_on: condition: service_healthy`. Secrets (unseal keys, AppRole secret IDs, DB passwords) are supplied via files in `./.secrets/` mounted as Docker secrets, not environment variables.

### 6.2 Two run modes
- `make dev`: OpenBao in dev mode (in-memory, auto-unsealed), plain HTTP inside the network, for fast iteration only.
- Normal mode (`docker compose up -d`): production-like single node with Raft storage and real initialization/unseal. This is the mode used for the demo and the docs. **Staging of TLS:** milestone M1 starts with plain HTTP on the private Docker network and the host port bound to 127.0.0.1 only; TLS for OpenBao and every inter-service hop is added as milestone **M1b**, before the gateway is finalised. Record this in `docs/decisions.md` and `docs/limitations.md` until M1b is done.
- OpenBao restarts **sealed**: after any restart the human must unseal with 3 of the 5 keys. Document this in `docs/setup.md`.

### 6.3 OpenBao bootstrap (`infra/openbao/bootstrap.sh`, idempotent)
Order of operations:
1. Initialise with the default 5 key shares and threshold 3 (`bao operator init`). The **human** copies the keys and root token to a location outside the workspace; scripts and agents never handle them. Print a warning that this is PoC-only; production should use auto-unseal or split key custody.
2. Unseal (human enters 3 keys; the prompt hides them; do not pipe keys in).
3. **Audit device is declarative:** it is defined in the server config file (`audit "file" "to-file" { options { file_path = "/openbao/logs/audit.log" } }`), not enabled via the API (`bao audit enable` fails with "use declarative, config-based audit device management instead"). Verify the log file receives entries after the first request.
4. Enable **KV v2** at `secret/`. Create canary paths `secret/canary/*` (section 11).
5. Enable **database** engine; configure connection to `pg-demo`; create:
   - dynamic role `app-readonly` (TTL 5 min, max 30 min, read-only grants);
   - static role `report-job` (rotation period 24 h).
   Rotate the engine's root DB credential so the original is no longer valid.
6. Enable **PKI**: generate root CA (internal), set URLs, create role `script-identity` (client certs only, EC P-256, TTL 1 h, max TTL 24 h, CN pattern `<identity>.keywarden.local`). *(Production note: use root offline + intermediate; for the PoC one CA is acceptable and must be documented.)*
7. Enable auth methods: **AppRole** (for gateway, policy-manager, and optional services). Cert auth is optional (P2 demo of direct cert login); the gateway does mTLS verification itself.
8. Create **token role** `script-token`: allowed policies glob `script-*`, orphan, non-renewable, short max TTL.
9. Create **password policy** `kw-strong` (length 32, at least 4 lowercase, 4 uppercase, 4 digits, 2 symbols). The gateway applies the same rules when generating KV secret values.
10. Write policies (below), create AppRoles, write role IDs and secret IDs to `./.secrets/`.
11. **Do not revoke the root token until a rescue path exists and has been tested.** In 2.6.x `operator generate-root` uses authenticated endpoints, so revoking the only root token without a rescue identity cannot be undone. Create a `root-rescue` policy limited to `sys/generate-root-token/*`, assign it to a `userpass` rescue user, test regenerating a root token with that user and the unseal keys, and only then revoke the root token. Document the procedure in `docs/theft-response.md` / `docs/setup.md`. Note that scripts in this bootstrap use an admin token supplied by the human at run time.

### 6.4 Policies (`infra/openbao/policies/`)
- `gateway.hcl`: may `create`/`update` on `auth/token/create/script-token`; read on `pki/cert/ca`, sign on `pki/sign/script-identity`, revoke on `pki/revoke`; `sys/leases/revoke-prefix/database/creds/*`; `database/rotate-role/*`; KV `secret/data/*` write/read **only** for rotation (new versions) and `secret/metadata/*` read/list. It must **not** be able to write policies.
- `policy-manager.hcl`: write/delete `sys/policies/acl/script-*` only. Used only by the policy-sync module when an admin changes an identity's allowed secrets.
- `script-<identity>` (generated): `read` on exactly the allowed `secret/data/<path>` and `database/creds/<role>`; nothing else. Canary paths are never included.
- `admin.hcl`: for human admin tasks (not used at runtime).

### 6.5 TLS (milestone M1b, after the first OpenBao milestone works over HTTP)
`infra/tls/make-dev-ca.sh` creates a development CA and server certificates for `openbao`, `gateway` (data and admin) and `decoy`. The gateway's data-plane listener requires a client certificate and verifies it against the **PKI CA** exported from OpenBao. Document how the dev CA differs from the PKI CA.

### 6.6 Environment variables (`.env.example`, placeholders only)
`KW_ENV`, `OPENBAO_ADDR`, `OPENBAO_CACERT`, `GATEWAY_APPROLE_ROLE_ID_FILE`, `GATEWAY_APPROLE_SECRET_ID_FILE`, `DATABASE_URL` (via secret file), `ML_URL`, `ML_TIMEOUT_MS=150`, `ML_FAIL_MODE=open`, `AUTH_MODE=dev|oidc`, `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `ADMIN_DEV_USER`, `DEMO_MODE=false`, `CANARY_WEBHOOK_HMAC_KEY_FILE`, `CERT_TTL=1h`, `LEARNING_MIN_EVENTS=50`, `RESPONSE_POLICY_FILE`, `LOG_LEVEL`.

---

## 7. Data model (PostgreSQL via Prisma)

All tables have `id` (uuid), `created_at`. Add indexes on the filtered/ordered columns.

| Table | Key fields |
|---|---|
| `identities` | `name` (unique, becomes cert CN), `type` (`script`/`ci`/`service`), `status` (`active`/`quarantined`/`revoked`), `allowed_secrets` (text[]), `max_rpm`, `owner`, `last_seen_at`, `current_cert_serial`, `cert_expires_at`, `learning` (bool), `ci_subject` (for JWT mapping) |
| `enrollment_tokens` | `identity_id`, `token_hash`, `expires_at`, `used_at` |
| `secrets_meta` | `path` (unique), `kind` (`kv`/`db_dynamic`/`db_static`), `sensitivity` (`low`/`medium`/`high`), `rotation_interval`, `last_rotated_at`, `expires_at`, `rotation_hook`, `status`, `compromised_in_history` (bool), `source` (`manual`/`migrated`) |
| `access_events` | `ts`, `request_id`, `identity_id`, `secret_path`, `action` (`read`/`dynamic`/`enroll`/`renew`/`denied`), `source_ip`, `user_agent`, `outcome`, `risk`, `risk_level`, `reasons` (jsonb), `decision`, `model_version`, `prev_hash`, `hash` |
| `alerts` | `ts`, `severity`, `type` (`anomaly`/`canary`/`bypass`/`degraded`/`expiry`), `identity_id`, `secret_path`, `event_id`, `reasons`, `status` (`open`/`ack`/`false_positive`/`resolved`), `actions_taken` |
| `response_actions` | `alert_id`, `action`, `target`, `status`, `detail`, `idempotency_key` |
| `rotations` | `secret_path`, `trigger` (`schedule`/`alert`/`manual`/`migration`), `hook`, `status`, `detail`, `ts` |
| `canaries` | `kind` (`api`/`ssh`/`vault`), `token_id` (public id), `token_hash`, `planted_in` (path/description), `status`, `last_hit_at` |
| `scan_reports` | `repo`, `ts`, `summary` (counts by type/severity) |
| `migration_items` | `report_id`, `file`, `line`, `rule`, `secret_path`, `status` (`found`/`imported`/`patched`/`rotated`), `patch_ref` (**never** the secret value) |
| `approvals` (P1) | `event_id`, `status`, `expires_at`, `decided_by` |
| `model_registry` | `version`, `trained_at`, `params`, `metrics`, `artifact_path`, `active` |
| `identity_baselines` | `identity_id`, `stats` (jsonb: hour histogram, rate, known IPs, known secrets, interarrival median/MAD) |

**Tamper-evident log (P1):** `hash = sha256(prev_hash || canonical_json(event))`. Provide `GET /admin/audit/verify` to recompute the chain and report the first mismatch. Use a DB-level rule/role so the app role cannot UPDATE/DELETE `access_events`.

---

## 8. Gateway API

Cross-cutting: JSON only, `Cache-Control: no-store`, request IDs (`X-Request-Id`), structured logs with redaction, global rate limit plus per-identity limit (`max_rpm`), body limit 64 KB, timeouts on all outbound calls.

### 8.1 Data plane (port 8443, mTLS; JWT for CI)

| Method & path | Auth | Purpose |
|---|---|---|
| `POST /v1/enroll` | one-time enrollment token (body) over TLS (no client cert yet) | Body: `{token, csr_pem}`. Verifies the token hash/expiry/unused, validates that the CSR's CN equals the identity name, signs through `pki/sign/script-identity`, returns `{certificate, ca_chain, expires_at}`. Marks token used. |
| `POST /v1/renew` | mTLS | Body: `{csr_pem}`. Same as enroll for an existing active identity. Must be called before expiry. |
| `GET /v1/secrets/:path` | mTLS or CI JWT | Fetch a static secret. Returns `{value, version, expires_at}`. Flow in 8.3. |
| `GET /v1/dynamic/:role` | mTLS or CI JWT | Dynamic DB credentials: `{username, password, lease_id, expires_at}`. |
| `GET /v1/whoami` | mTLS | Identity name, status, cert expiry (no secret info). |
| `GET /v1/health` | none | Liveness only. |

Errors: `401` bad/absent identity; `403` not allowed / denied (generic message, identical for "not permitted" and "blocked by risk"); `404` unknown secret (same shape as 403 for identities that have no ACL, to avoid enumeration); `429` rate limited; `503` dependency down. Never reveal why.

### 8.2 Control plane (port 8444, admin auth: dev token or OIDC JWT; all state-changing calls are audited)

- Identities: `POST /admin/identities`, `GET /admin/identities`, `PATCH /admin/identities/:id` (ACL, limits), `POST /admin/identities/:id/enrollment-token`, `POST /admin/identities/:id/quarantine`, `POST /admin/identities/:id/release`, `DELETE /admin/identities/:id` (revokes cert).
- Secrets: `POST /admin/secrets` (create/import, sensitivity, rotation, hook), `GET /admin/secrets` (metadata only, **never values**), `POST /admin/secrets/:path/rotate`, `PATCH /admin/secrets/:path`.
- Events/alerts: `GET /admin/events` (filters: identity, secret, time, risk, outcome; CSV export), `GET /admin/alerts`, `PATCH /admin/alerts/:id` (ack / false positive / resolve, feedback to ML), `GET /admin/stream` (SSE: events, alerts, actions).
- Rotation: `GET /admin/rotations`.
- Canaries: `POST /admin/canaries`, `GET /admin/canaries`; internal `POST /internal/canary-hit` (HMAC-signed, from the decoy service).
- Scan/migration: `POST /admin/scan-reports` (upload JSON from the scanner, values redacted), `GET /admin/scan-reports`, `POST /admin/migration/import`.
- Audit: `GET /admin/audit/verify` (P1).
- Model: `GET /admin/model` (proxy to ML info), `POST /admin/model/train`.
- Demo (only when `DEMO_MODE=true`): `POST /admin/demo/simulate` with `{scenario}`.
- Health/metrics: `GET /admin/health` (dependencies), `GET /admin/metrics` (counters, p95 latency).

### 8.3 Secret fetch flow (the core sequence)
1. TLS handshake with required client cert. Verify chain against the PKI CA. Extract CN. Reject expired certificates.
2. Load identity. Reject if `quarantined`/`revoked` (record a denied event). Check cert serial is not in the revoked set.
3. Rate limit.
4. Authorize: secret is in `allowed_secrets`. If the path is a **canary** → emit critical canary alert and respond 403 (see section 11).
5. Check secret status/expiry (expired → deny and alert).
6. Build the event (`identity`, `secret_path`, `sensitivity`, `source_ip`, `user_agent`, `ts`, `action`), call ML `/score` (timeout 150 ms). On failure apply D8.
7. Response engine decides (section 10).
8. If allowed: request a token `auth/token/create/script-token` with policy `script-<identity>`, TTL 30 s, `num_uses` 1, metadata `{identity, request_id}`; read the secret with it; return the value; the token expires/uses up immediately.
9. Persist the event (with decision, risk, reasons, hash chain) **before** returning. Execute asynchronous actions (alerts, rotation, quarantine) idempotently.

**Bypass detector:** a background job tails the OpenBao audit log. Any request on `secret/data/*`, `secret/metadata/*` or `database/creds/*` whose token metadata does not carry a gateway `request_id` (and is not on a small allowlist for bootstrap/admin) raises a `bypass` alert. Inspect real audit log lines to write the parser; field names may differ by version.

### 8.4 Enrollment ("secret zero") flow
Admin creates an identity and an enrollment token (shown once, TTL 10 min, single use, only the hash is stored) → delivered out-of-band → script host runs `keywarden enroll --token ...` → CLI generates key pair locally, sends CSR → gateway signs → cert stored with 0600 permissions. From then on, authentication is by certificate only. Document this clearly: enrollment tokens are the only bootstrap secret and are short-lived and single-use.

---

## 9. ML service (FastAPI, `services/ml/`)

### 9.1 Endpoints
- `POST /score`: input event → `{risk: 0-100, level: low|medium|high|critical, reasons: [str], model_version, mode: "model"|"rules"|"learning"}`. Target p95 latency under 50 ms.
- `POST /train`: train from baseline events in DB (or a provided dataset); registers a new model version; does not activate unless metrics gates pass.
- `GET /model`: active version, trained_at, parameters, metrics.
- `POST /feedback`: analyst labels (false positive / confirmed) stored for evaluation and retraining.
- `GET /health`.

### 9.2 Event schema (input)
`event_id, ts, identity, secret_path, sensitivity, source_ip, user_agent, action, success`. History features are computed by the service from the database using a read-only role and an in-memory baseline cache (refresh every few minutes).

### 9.3 Features (per event, identity-relative where possible)
- Time: `hour_sin`, `hour_cos`, `is_weekend`, deviation of hour from the identity's usual hour distribution.
- Rate: requests in last 1 min / 10 min / 1 h; ratio to the identity's typical rate.
- Breadth: distinct secrets in last 10 min / 1 h; `is_new_secret_for_identity`.
- Source: `is_new_source_ip`, count of distinct source IPs in the last day.
- Rhythm: seconds since the identity's last event, z-score against its median interarrival (robust: median/MAD). Scripts are periodic, so this is a strong signal.
- Sensitivity ordinal (0,1,2).
- Failures: failed/denied attempts in the last 5 min.

### 9.4 Model and scoring
- **Isolation Forest** (`n_estimators=200`, fixed `random_state`) trained on baseline "normal" events only. Convert `-decision_function` to a 0–100 risk using the percentile rank within the training score distribution.
- **Rules** (explainable, weighted): new source IP on medium/high sensitivity; hour far outside the identity's usual window; rate more than N× baseline; never-before-seen secret; ≥5 failures in 5 min; canary → 100.
- **Combine:** `risk = max(model_risk, rule_risk)`. Levels: low <40, medium 40–69, high 70–84, critical ≥85 (configurable in the response policy).
- **Learning mode:** identities with fewer than `LEARNING_MIN_EVENTS` baseline events use rules only with conservative thresholds, are marked `learning`, and are never auto-quarantined (alerts only).
- **Explanations:** for each of the top 3 deviating features compute the robust z-score against the identity baseline, and render a template such as: `Accessed at 03:12 (usual window 01:50–02:20)`, `14× the normal request rate`, `Source IP never seen for this identity`. Return these as `reasons`.
- **Baseline hygiene:** retrain/update baselines only from events with risk below medium and no alert; use a bounded window (for example the last 30 days). Version every model; keep the previous one for rollback.

### 9.5 Synthetic data generator (`ml-data/generate.py`)
No public dataset of vault access logs exists, so generate one. Seeded and deterministic.
- **Identities (about 20) from archetypes:** `cron_5min` (every 5 min ±10 s, 24/7), `hourly_report` (minute 5 of each hour), `nightly_backup` (02:00 daily, 2–3 secrets within 30 s), `ci_pipeline` (weekday 08:00–19:00 bursts, Poisson rate, pool of 4 CI IPs), `admin_tool` (sporadic business hours, 1–3 secrets, 2 IPs). Randomise parameters per identity.
- **Secrets:** about 25, mixed sensitivity.
- **Duration:** 30 days. **Split by time**: days 1–21 train, days 22–30 test (no random shuffling, to avoid leakage).
- **Injected attack scenarios (labelled, test period only):** (1) off-hours access with valid identity; (2) burst/bulk pull; (3) new source IP with a valid cert; (4) lateral movement to secrets never used by that identity; (5) failed-auth spike; (6) canary read; (7) **slow-and-low** (small deviations over days); (8) **mimicry** (attacker copies the victim's schedule but from a new IP). Scenarios 7 and 8 are deliberately hard.
- Output: CSV/Parquet with labels `is_attack`, `scenario`.

### 9.6 Evaluation protocol (`services/ml/eval/`)
Compare three detectors on the same test set: (a) static thresholds only, (b) Isolation Forest only, (c) hybrid (ours). Report precision, recall, F1, ROC-AUC, PR-AUC, **false positives per identity per day** (the operational metric), per-scenario recall, and **detection latency** (events or seconds until the first alert in an attack sequence). Include an ablation of feature groups. Save figures and a results table to `docs/ml-evaluation.md` and `docs/ml/`. State plainly that results are on synthetic data and are likely optimistic, and report the weak scenarios (7 and 8) honestly. Seeds and commands must be in the doc so results are reproducible.

---

## 10. Response engine and rotation

### 10.1 Policy file (`response-policy.yaml`, hot-reloadable)
```yaml
thresholds: { alert: 40, step_up: 70, block: 85 }
sensitivity_adjust: { low: 0, medium: 0, high: -10 }   # high-sensitivity secrets trigger earlier
levels:
  low:      [log]
  medium:   [alert_info]
  high:     [alert_high, mark_rotation_due]            # P1: require_approval
  critical: [deny, alert_critical, rotate_secret, quarantine_identity]
canary_hit: [alert_critical, quarantine_source_identity_if_known]
learning_mode_cap: high        # learning identities are never auto-quarantined
cooldown_seconds: 300          # suppress duplicate alerts per identity+type
```

### 10.2 Actions (each idempotent, recorded in `response_actions`)
- `alert_*`: write alert, push SSE, optional webhook (Slack/email stub).
- `deny`: return generic 403.
- `rotate_secret`: run the secret's rotation hook (10.3). For dynamic DB roles, revoke the lease prefix so all issued credentials die.
- `quarantine_identity`: set status `quarantined`; revoke the certificate serial via PKI; revoke outstanding tokens/leases; the gateway denies all further requests. `release` requires an admin and a fresh enrollment token (re-enrol, new key pair).
- `mark_rotation_due`: flags the secret; the scheduler rotates within the configured window.
- `require_approval` (P1): the request returns `202 {approval_id}`; the CLI waits up to `--wait` seconds while an admin approves in the dashboard (TTL 5 min, then auto-deny).

### 10.3 Rotation hooks (interface `RotationHook { name; rotate(ctx): Result; verify(ctx): Result }`)
- `postgres-static`: calls OpenBao `database/rotate-role/<role>`.
- `postgres-dynamic-revoke`: revokes lease prefix for the role.
- `webhook`: POSTs to a configured URL to update the target system; verifies with a follow-up check.
- `kv-random-only`: generates a new strong value (same rules as `kw-strong`), writes a new KV version, and marks the secret **"manual follow-up required: target not updated"** (never claim success silently).

### 10.4 Scheduling and expiry
`@nestjs/schedule` jobs: rotate secrets older than `rotation_interval`; alert 24 h before `expires_at`; after `expires_at` the gateway denies access and raises an `expiry` alert; renew-or-warn for identity certificates; nightly baseline refresh and optional retraining.

---

## 11. Canary (decoy) system

Goal: a near-zero false-positive signal that also tells us **which leaked file** was used.

- **Vault canaries:** secrets under `secret/canary/*` (for example `prod-db-admin`, `backup-ssh-key`). No identity policy ever allows them. Any request for these paths through the gateway, and any attempt seen in the OpenBao audit log (including denied ones), is a critical alert.
- **Planted canaries:** `keywarden canary create --kind api|ssh|db --plant <file>` generates a unique decoy credential (`kwc_<kind>_<random>`), writes it into the given legacy script in the same style as a real hardcoded secret, and registers it with `planted_in`. Store only the hash, plus a public `token_id` to look it up.
- **Decoy API (P0):** a FastAPI service that looks like an internal API. Any request carrying a `kwc_` token (header, query or body) is reported to the gateway through `POST /internal/canary-hit` (HMAC-signed, includes source IP, user agent, `token_id`). The decoy service always returns a plausible 401/200 to avoid revealing itself, and must be network-isolated from internal services.
- **SSH honeypot (P2):** `asyncssh` server that logs any login attempt using a decoy username/password and always denies.
- **Response:** `canary_hit` → critical alert showing which file/location the credential was planted in; if the hit maps to a known identity, quarantine it; always add the source IP to the dashboard's "watch" list.
- Do **not** reuse real-looking provider key formats.

---

## 12. Scanner and migration engine (`scanner/`)

### 12.1 Discover
- Run **Gitleaks** (verify the installed version's CLI: newer versions use `gitleaks dir` / `gitleaks git`; older use `detect`) with JSON output, plus a custom `.gitleaks.toml` adding generic rules for assignments such as `password`, `passwd`, `pwd`, `secret`, `token`, `api_key`, `apikey`, `connection string`, in Python, Bash, PowerShell, JS/TS, `.env`, YAML, `.properties`, and PEM private-key blocks.
- Scan working tree and (optionally) git history.
- Parse findings into `{file, line, rule, kind, entropy, fingerprint}`. **Redact values** in every report and log. Classify type (password, API key, DB URL, private key) and suggest sensitivity.

### 12.2 Migrate
`keywarden migrate import --report report.json --prefix <team/app>`:
1. For each finding, store the value in KV at `secret/migrated/<repo>/<name>` (read from the file at import time, not from the report), set sensitivity and metadata, mark `compromised_in_history=true` if found in git history.
2. Propose an identity and ACL for the script (admin confirms in dashboard or CLI).
3. Trigger rotation through a hook where available, because **the old value is already exposed in code and history**. If no hook exists, flag "rotate at source" in the migration report.
4. Never write secret values into patches, reports, logs or commit messages.

### 12.3 Patch generator
Produces a unified diff (default **dry-run**); `--apply` creates a new git branch and writes the changes; never auto-commits to the default branch. Strategies by language:
- **Python:** use `libcst` to replace `NAME = "literal"` with `NAME = keywarden.get("<path>")` and add the import. Handle f-strings and dict values conservatively; if unsure, leave a `# TODO(keywarden)` marker and report it.
- **Bash:** `NAME="literal"` → `NAME="$(keywarden get <path>)"`. Prefer `keywarden run --env NAME=<path> -- ./script.sh` when the script reads env variables.
- **PowerShell:** `$Name = "literal"` → `$Name = Get-KWSecret -Name '<path>'`; add `Import-Module KeyWarden`.
- **JS/TS:** `const X = "literal"` → `const X = await kw.get("<path>")` (or env injection via `keywarden run`).
- **.env / YAML / properties:** replace the value with a placeholder and document using `keywarden run --env-file`.
- Idempotent (running twice changes nothing), preserves formatting, and has golden-file tests (`demo/legacy-repo` → `demo/fixed-repo`).
- After patching, print guidance for cleaning git history (`git filter-repo` or BFG) and a reminder that cleaning history does **not** make the old secret safe unless it has been rotated.

### 12.4 Reports
`scan_reports` summary (counts by type/severity/file), migration status per finding, and an exportable Markdown/PDF report for the demo.

---

## 13. CLI and client libraries

### 13.1 CLI (`keywarden`, TypeScript)
| Command | Purpose |
|---|---|
| `keywarden enroll --token T [--gateway URL --name N]` | Generate key pair, send CSR, store cert |
| `keywarden renew` | Renew certificate (mTLS) |
| `keywarden agent [--interval]` | Long-running renewal daemon (systemd/Windows service friendly) |
| `keywarden get <path> [--json\|--raw\|--env NAME]` | Fetch a static secret. Raw output suitable for `$(...)` |
| `keywarden dynamic <role> [--json]` | Fetch dynamic DB credentials |
| `keywarden run --secret NAME=path ... -- <cmd>` | Inject secrets as environment variables into a child process without writing to disk; supports `--env-file` |
| `keywarden whoami` / `status` | Identity and cert info |
| `keywarden canary create ...` | Section 11 |
| `keywarden scan <dir>` / `migrate import` / `migrate patch` | Section 12 |

- Config: `~/.keywarden/config.json` (gateway URL, CA bundle path), key and cert in `~/.keywarden/` with `0600`; on Windows restrict the ACL to the current user. Support `KEYWARDEN_HOME`.
- Exit codes: `0` ok; `2` usage; `3` auth failure; `4` forbidden; `5` not found; `6` rate limited; `7` gateway unavailable. Never print secrets to stderr or logs; mask in `--verbose`. Secrets go to stdout only when explicitly requested.
- Retries with backoff for `503`/network errors; do not retry on `403`.
- P2: package as a single binary (Node SEA or equivalent) for Linux and Windows.

### 13.2 Libraries
- **Python (`clients/python`)**: `keywarden.get(path)`, `keywarden.dynamic(role)`, uses the CLI's cert files; minimal dependencies.
- **PowerShell (`clients/powershell`)**: `Invoke-KWEnroll`, `Get-KWSecret`, `Get-KWDynamicCredential`, `Update-KWCertificate`; works on PowerShell 5.1 and 7; Pester tests.

---

## 14. Dashboard (React + Vite + TypeScript)

**Design:** dark theme by default (light optional), consistent severity colours (low = green, medium = amber, high = orange, critical = red) that also use icons/labels (do not rely on colour alone), good contrast (WCAG AA), keyboard-navigable, responsive down to tablet width, skeleton loaders and helpful empty states. Clean and calm: this is a security console.

| Page | Content |
|---|---|
| Overview | KPI cards (secrets managed, hardcoded found vs migrated, active identities, alerts in 24 h, average risk), live event feed (SSE), risk-over-time chart, top risky identities |
| Audit trail | Filterable, searchable table (identity, secret, time, outcome, risk), detail drawer with reasons and decision, CSV export, "verify integrity" button (P1) |
| Alerts | List with severity, reasons, actions taken; acknowledge / mark false positive / resolve; canary alerts show the planted location |
| Identities | Status, cert expiry countdown, last seen, ACL editor, quarantine/release, enrollment token generator (shown once) |
| Secrets | Metadata only (never values): sensitivity, rotation schedule, last rotated, expiry, hook, "manual follow-up required" badges; rotate-now |
| Rotation | History timeline and upcoming schedule |
| Migration | Scan report summary, findings by type/severity, diff viewer for generated patches (values redacted), status per finding |
| Canaries | Planted decoys, locations, hits |
| Model | Active version, metrics, last trained, drift/learning-mode identities |
| **Demo (when `DEMO_MODE`)** | **Attack simulator**: buttons for off-hours, burst, new IP, lateral movement, canary hit, with a live result panel showing score, reasons and the response taken. This page makes the live demo reliable |

Use TanStack Query for data, SSE for live updates, Recharts for charts. Typed API client generated from the gateway's OpenAPI document (NestJS Swagger).

---

## 15. DevOps and platform integration

- **Linux:** `systemd` unit for `keywarden agent`; example `cron` job and `systemd` timer that call `keywarden run`/`get`; package as a `.deb` (P2).
- **Windows:** PowerShell module; example Task Scheduler job that uses `Get-KWSecret`; document that the private key file is ACL-restricted; Windows Certificate Store with non-exportable keys is a documented upgrade path (P2).
- **Cloud:** run the full stack on one small GCP VM with Docker Compose. Only 8443/8444 (and the decoy port) are reachable; OpenBao and databases are never exposed. Write `infra/gcp/vm-setup.md` and `firewall.md`.
- **CI/CD (P1):** `.github/workflows/demo-oidc.yml` uses GitHub Actions OIDC (permissions `id-token: write`). The job requests an OIDC token for the audience `keywarden`, and calls the gateway with it. The gateway validates the JWT signature via GitHub's JWKS, checks `iss`, `aud`, `repository` and `ref` against the identity's `ci_subject`, and then follows the normal fetch flow, so **no stored token or secret exists in the workflow**. For local development without a public endpoint, use a locally signed test JWT and a mock JWKS (documented as a test double, not as the real flow).
- **Containers:** example of `keywarden run` as an entrypoint wrapper in a Dockerfile; Kubernetes is out of scope (mention under future work).
- **Upgrade paths (document only):** TPM-backed or HSM-backed key storage for the CA key and client keys; OpenBao auto-unseal.

---

## 16. Testing and acceptance

### 16.1 Layers
Unit tests (Jest, pytest, Pester), integration tests against Compose (`make test-int`), end-to-end scenarios (`make e2e`), security tests, performance measurements. CI runs lint, type checks, unit tests, gitleaks and the licence check.

### 16.2 Required end-to-end and security scenarios
1. Enrol → fetch secret → value returned; no secret appears in logs.
2. Expired cert → 401. Revoked/quarantined identity → denied.
3. Identity requests a secret outside its ACL → 403 and event recorded.
4. Fetch without a client cert → rejected.
5. Replay of a used enrollment token → rejected. Expired enrollment token → rejected.
6. Direct request to OpenBao from outside the internal network is impossible (network test); a direct access attempt with a stolen token is flagged as bypass.
7. Dynamic credentials work against `pg-demo` and stop working after TTL or lease revocation.
8. Static DB role rotation changes the password and the platform keeps working.
9. Normal traffic produces no alerts above medium after the learning period.
10. Each injected attack scenario produces the expected alert, reasons and response.
11. Canary hit (vault path and decoy API) produces a critical alert within 2 seconds and identifies the planted location.
12. ML service down → fallback mode works and a "degraded detection" alert is raised; high-sensitivity secrets fail closed.
13. Patch generator is idempotent and matches the golden files; patches never contain secret values.
14. Audit hash chain verifies; modifying a row is detected (P1).
15. Scanner finds all planted secrets in `demo/legacy-repo` with zero values in the report.

### 16.3 Measurements to report (measure; do not assume)
Retrieval latency overhead (p50/p95) with and without scoring; ML scoring p95; CPU and memory of the full stack on a small VM (`docker stats`); migration coverage (found vs migrated in the demo repo); ML metrics from section 9.6. Targets to aim for: ML scoring p95 < 50 ms, end-to-end fetch p95 < 250 ms on a small VM, full stack within a few GB of RAM. Report actual numbers even if they miss the targets.

---

## 17. Documentation deliverables (`docs/`)

The competition asks for documentation explaining how everything is set up, including MFA, password complexity and credential theft.

| File | Must contain |
|---|---|
| `architecture.md` | Diagram, components, data flows, design decisions summary |
| `setup.md` | From a clean machine to a working stack, exact commands, expected output, troubleshooting |
| `authentication.md` | Certificate-based identity design, enrollment, renewal, revocation; **comparison of SSH keys vs certificates vs TPM/HSM** (pros/cons) and why short-lived certificates were chosen; TPM/HSM upgrade path |
| `mfa.md` | Admin MFA with OIDC + TOTP (Keycloak setup steps), what is and is not protected by MFA (humans yes, machines use certificates) |
| `password-policy.md` | Secret complexity rules (`kw-strong`), generation, minimum lengths, rotation intervals by sensitivity |
| `rotation.md` | Hooks, schedules, expiry, dynamic credentials, what "manual follow-up" means |
| `theft-response.md` | Runbook for suspected credential theft: detect, contain (quarantine/revoke), eradicate (rotate), recover (re-enrol), review (audit trail) |
| `audit-schema.md` | Event fields, hash chain, retention, example queries |
| `migration-guide.md` | How a team moves a script from hardcoded to KeyWarden, including git-history cleanup and the rotate-after-migrate rule |
| `ml-evaluation.md` | Section 9.6 results and honest limitations |
| `threat-model.md` | Section 3.3 expanded, residual risks |
| `limitations.md` | Honest list of limits and non-goals |
| `licenses.md` | Every dependency and its licence |
| `decisions.md` | ADRs (D1–D10 plus any new) |
| `demo-script.md` | Section 18 with exact commands and expected output, plus a fallback plan if something fails live |
| `progress.md` | Milestone reports |

Also write a good `README.md`: what it is, a 5-command quick start, architecture picture, demo GIF/screenshots placeholders, links to docs.

---

## 18. Demo script (about 5 minutes; automate with `make demo`)

1. **Before:** show `demo/legacy-repo` with hardcoded secrets. Run `keywarden scan` → findings table (values redacted).
2. **Migrate:** `keywarden migrate import` then `migrate patch` → show the diff; the script now calls `keywarden get`. Show the old DB password was rotated.
3. **Protect:** run the script: it works. Show `keywarden whoami` (cert expires in under an hour) and a dynamic DB credential that expires in minutes.
4. **Normal behaviour:** a few normal accesses → dashboard shows low risk, no alerts.
5. **Attack:** from the Demo page trigger "off-hours burst from a new IP" → score, plain-language reasons, response (denied, secret rotated, identity quarantined) appear live.
6. **Canary:** use a planted decoy credential against the decoy API → instant critical alert naming the file it was planted in.
7. **Audit and integrity:** show the audit trail, the rotation history and the integrity check (P1).
8. **CI (P1):** show the GitHub Actions run fetching a secret with no stored token.
9. Close with numbers from section 16.3 and the honest limitations slide.

Prepare a fallback: pre-recorded terminal session and screenshots in case of network issues.

---

## 19. Milestones and build order

Each milestone has a Definition of Done (DoD). Do not skip the tests.

| # | Milestone | Priority | DoD |
|---|---|---|---|
| M0 | Scaffolding | P0 | Monorepo layout, Makefile, lint/format, CI skeleton, gitleaks hook, licence file, empty services boot in Compose |
| M1 | Core infrastructure | P0 | OpenBao (Raft, pinned 2.6.2) up over HTTP on the private network with real init/unseal done by the human; declarative audit log working; KV, DB engine (dynamic + static roles), PKI role, token role, policies, password policy, canary paths; `pg-platform` and `pg-demo`; root-token rescue path tested; stack starts from clean with the documented `docker compose` commands |
| M1b | TLS everywhere | P0 | Dev CA and certificates; OpenBao and all inter-service traffic on TLS; docs updated |
| M2 | Gateway (data plane) | P0 | mTLS identity, enrollment (CSR), renew, secret fetch via scoped single-use token, dynamic creds, event persistence, rate limiting, redaction; scenarios 1–5, 7 pass |
| M3 | CLI + demo scripts | P0 | `enroll/renew/get/dynamic/run/whoami/agent`; `demo/legacy-repo` and a working "after" version; Linux works |
| M4 | ML service and evaluation | P0 | Generator, features, model, `/score` with reasons, fallback rules, learning mode, evaluation doc with real numbers |
| M5 | Response engine, rotation, quarantine | P0 | Policy file, actions, hooks (`postgres-static`, `postgres-dynamic-revoke`, `webhook`, `kv-random-only`), scheduler, expiry, quarantine/release; scenarios 8, 9, 10, 12 pass |
| M6 | Canaries | P0 | Vault canaries, planted canaries, decoy API, HMAC webhook, bypass detector; scenarios 6, 11 pass |
| M7 | Dashboard (basic → full) | P0 basic, P1 full | Overview, Audit, Alerts, Identities, Secrets, Demo/simulator first; others after |
| M8 | Scanner and migration | P1 | Gitleaks wrapper, custom rules, import, patch generator for Python/Bash/PowerShell, golden tests; scenarios 13, 15 pass |
| M9 | Integrations | P1 | PowerShell module, systemd units, Task Scheduler example, GCP VM guide, GitHub Actions OIDC flow |
| M10 | Hardening and extras | P1/P2 | Hash-chained audit + verify, approval flow, Keycloak OIDC + TOTP MFA, SSH honeypot, single-binary CLI, Node/JS patching |
| M11 | Docs, measurements, rehearsal | P0 | All docs in section 17 complete with real numbers; `make demo` runs the full script; fallback recording prepared |

Suggested order if time is short: M0 → M1 → M2 → M3 → M4 → M5 → M6 → minimal M7 → M11, then M8 → M9 → M10.

---

## 20. Coding standards

- **TypeScript:** strict mode, ESLint + Prettier, no `any` without a comment, DTO validation on every endpoint, dependency injection, small modules, no business logic in controllers.
- **Python:** type hints, ruff + black, pytest, Pydantic models for I/O.
- **Errors:** typed errors mapped to HTTP codes; no stack traces to clients; log with request IDs.
- **Logging:** JSON logs (`pino`), redaction on, no secrets, levels via env.
- **Config:** 12-factor; fail fast on missing required config; sensible defaults for dev only.
- **Time:** store UTC; handle clock skew (cert `notBefore` tolerance of a few seconds).
- **Idempotency:** response actions, bootstrap script, migrations and patches must be safe to run twice.
- **Docs in code:** concise comments for security-relevant decisions; OpenAPI annotations on all endpoints.
- **Git:** feature branches, conventional commits, update `CHANGELOG.md`.

---

## 21. Things to verify against current docs (do not assume)

1. **OpenBao:** exact image name and tag, binary name (`bao`), environment variable names, audit-device enable syntax for the pinned version, token header accepted by the HTTP API (the Vault-compatible `X-Vault-Token` header or an OpenBao equivalent), PKI/database plugin names, token-role parameters, and the audit log field layout (parse real lines).
2. **Gitleaks:** subcommands and flags for the installed version; custom rule syntax.
3. **Node mTLS:** how to read and verify the peer certificate (`request.socket.getPeerCertificate()`), and TLS settings for `requestCert`/`rejectUnauthorized`; behaviour of the enrollment route that must be reachable **without** a client certificate (use a separate route handling or listener setting and test it).
4. **GitHub OIDC:** token claims, audience configuration and JWKS URL.
5. **Keycloak:** realm setup for OIDC with TOTP.
6. **PostgreSQL:** SQL for dynamic role creation and least-privilege grants in the DB engine statements.
7. **Licences:** confirm the licence of every dependency and Docker image before adding.
8. **Versions:** record every version used. The OpenBao image is pinned to 2.6.2 on purpose; before upgrading (for example to 2.7.x), read that release's notes: 2.7.0 removes the `file` storage backend and several built-in KMS seals, and changes the image's volume declarations. Always use Raft storage and named volumes.
9. **Shell portability:** the project owner may be on Windows. Do not assume bash on the host. Any `.sh` script runs inside a Linux container and must have LF line endings (add `.gitattributes` with `*.sh text eol=lf`). Document PowerShell and bash variants where host commands are needed.

If any of these contradicts this file, follow the official documentation, then record the change in `docs/decisions.md`.

---

## 22. Glossary (plain language, for the presenter)

- **Hardcoded credential:** a password or key written directly in a script.
- **Vault / secrets manager:** a protected service that stores secrets and hands them out only to approved callers.
- **OpenBao:** the open-source vault used as the secure core.
- **Certificate-based identity:** a short-lived digital ID card a script shows to prove who it is, instead of a password.
- **mTLS:** both sides of a connection prove who they are with certificates.
- **CSR:** a request to sign a certificate; the private key stays on the script's machine.
- **Dynamic secret:** a temporary credential created on demand that expires by itself.
- **Rotation:** replacing a secret with a new one so the old one stops working.
- **Audit trail:** a record of who accessed what, and when.
- **Isolation Forest:** an ML method that finds events that look unusual compared with normal behaviour.
- **Canary / honeytoken:** a fake credential that nobody should use; if someone does, we know something is wrong.
- **Quarantine:** blocking an identity until an admin re-approves it.
- **HSM / TPM:** hardware that protects keys physically; stronger but costlier or hardware-bound, so documented as an upgrade path.