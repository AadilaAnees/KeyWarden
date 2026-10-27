# Setup log

Record of what was done, when, and the result. Never write keys, tokens or passwords here.

## Environment
- Operating system: Windows
- Docker Desktop version: 29.6.1
- Terminal: PowerShell in VS Code
- OpenBao image: openbao/openbao:2.6.2

## Decisions made
- OpenBao instead of HashiCorp Vault, because Vault's licence is not OSI open source.
- Raft storage on a named Docker volume (the file backend is being removed upstream).
- Audit device is set in the server config file, not enabled by command.
- Plain HTTP for now, port bound to 127.0.0.1 only. TLS will be added in milestone M1b.
- Unseal keys and root token are stored outside the project folder, and are not in Git.
- Root token is NOT revoked yet. This waits until a rescue path is created and tested.

## 23.10.2026 M0: repository scaffold
- Created project folder and ran `git init`.
- Added .gitignore, .gitattributes, README.md, AGENTS.md, docs/ and infra/.
- Result: done.

## [23.10.2026 Problem: Docker Desktop engine would not start
- Symptom: `docker compose up -d` failed with "Docker Desktop is unable to start".
- What fixed it: wsl install

## 27.10.2026 M1 part 1: OpenBao core
- Started OpenBao with `docker compose up -d`. Result: container running.
- Initialised (5 key shares, threshold 3) and unsealed. `bao status` showed Sealed false, Storage Type raft.
- Enabled the KV v2 engine at `secret/`, stored and read back a test secret (fake value).
- Audit log: audit.log exists and is receiving entries. Check: tokens and values appear hashed, not in plain text. [write the two grep results]
- Restart test: after `docker compose restart openbao` it came back sealed. After unsealing, the test secret was still there.
- Result: done.

## Status
- M0: complete.
- M1: in progress. Part 1 done. Remaining: demo Postgres and dynamic credentials, password policy, PKI, policies and AppRole, canary paths, root-token rescue, then TLS (M1b).
