---
name: secrets-rotation
description: Secrets and API-key management for automation and AI projects (verified 2026-10-03) — where secrets live (vault vs CI secrets vs env), rotation strategies (scheduled, dual-key overlap, dynamic/ephemeral, OIDC instead of static keys), a zero-downtime rotation runbook, leaked-key incident response (revoke → rotate → scrub → audit, in that order), pre-commit and history scanning, special cases (LLM provider keys, n8n N8N_ENCRYPTION_KEY, webhook HMAC secrets, Telegram bot tokens, OAuth refresh tokens, DB passwords), and agent-specific rules (never in prompts, per-agent credentials). Use when setting up a new project's secrets, before publishing a repo, when a key leaks or an employee/contractor leaves, when planning periodic key rotation, or when wiring credentials into agents, n8n, Docker or CI. Pairs with ai-agent-security and portable-skills (layered secret lookup).
---

# Secrets & key rotation

Principles: **short-lived beats rotated, rotated beats static; scoped beats shared; revocable in one step.** Every secret has an owner, a scope, a storage location, and a rotation plan written down before it's created.

## 1. Where secrets live

| Option | Use for | Notes |
|---|---|---|
| **Secrets manager** (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, HashiCorp Vault, Infisical, 1Password/Bitwarden Secrets) | Anything long-lived or shared, production | Audit log, access policy, built-in rotation; preferred |
| **CI secrets** (GitHub Actions secrets/environments, GitLab CI variables) | Short-lived, low-blast-radius CI credentials | Better: OIDC federation so CI holds *no* cloud keys at all |
| **Runtime env vars** from the manager / orchestrator | App consumption | Inject at runtime; never bake into images |
| **`.env` file** | Local dev only | In `.gitignore` from the first commit; separate dev keys with low limits |
| **OS keychain** (macOS Keychain, `secret-tool`) | Personal CLI tools, scripts | Layered lookup: env → keychain → file (see `portable-skills`) |
| ❌ Source code, Docker images, n8n workflow JSON, prompts, Slack/Telegram messages, tickets | Never | |

**OIDC first:** GitHub Actions → AWS/Azure/GCP via OIDC federation gives a 15–60 min token per job; nothing to rotate or leak. Use it wherever the target supports it (cloud providers, Vault, npm trusted publishing).

## 2. Rotation strategies

- **Dynamic / ephemeral:** credentials issued per app start or per job, expire automatically (Vault DB engine, cloud IAM role credentials, OIDC). Best when available.
- **Scheduled automated rotation:** the manager rotates on a timer using the provider's built-in rotation (prefer built-in over custom Lambdas — fewer misconfigurations). Custom rotation = 4 steps: **create → set → test → finish**.
- **Dual-key overlap (zero downtime):** two valid keys at once. Deploy consumers with the new key while the old still works, verify, then revoke the old. For signing keys: sign with new, verify with both until old tokens expire.
- **Rotate on event, always:** suspected leak, person with access leaves, vendor breach, key seen in logs/screens/screenshots, repo made public, laptop lost.

Suggested cadence (adjust to risk): session/OIDC tokens minutes–hours · CI and service API keys 30–90 days or on event · DB passwords 90 days or dynamic · KMS data keys yearly (automatic) · webhook HMAC secrets on event + yearly. Human passwords: on compromise only (NIST), plus MFA.

## 3. Zero-downtime rotation runbook

1. **Inventory:** where is this secret used? (grep configs, CI, n8n credentials, servers, partner integrations). Missing one consumer = outage.
2. **Create** the new key with the same or narrower scope; label it with date + purpose (`prod-n8n-2026-10`).
3. **Distribute** to every consumer via the secrets manager / CI / n8n credential (update in place so references stay valid).
4. **Restart/reload** consumers; **test** a real call per consumer (not just "deploy green").
5. **Watch** the old key's last-used timestamp (most providers show it) until it stops moving.
6. **Revoke** the old key. Record date, who, why.
7. **Update** the inventory and set the next rotation reminder.

## 4. Leaked secret — incident response

Order matters: **revoke/rotate first, clean up second.** Removing a key from git does nothing if it's still valid — bots scrape public GitHub within minutes.

1. **Revoke** the exposed key immediately (or rotate if revocation would cause an outage you can't take — then revoke within the hour).
2. **Rotate:** issue a new key, deploy to consumers (runbook above).
3. **Assess:** provider usage logs / billing / audit log for the exposure window — unknown IPs, unusual models, spend spikes, data access. LLM keys get abused for resale within hours.
4. **Scrub:** remove from code *and history* (`git filter-repo` or BFG), force-push, ask GitHub support to purge cached views/PR refs if public; delete from logs, CI output, tickets, chat.
5. **Find the cause:** how did it get there (committed .env, debug log, screenshot, AI-generated code with a pasted key)? Fix the cause — add the pre-commit hook, the `.gitignore` entry, the log redaction.
6. **Personal data involved?** If the key gave access to personal data and misuse can't be ruled out → GDPR breach assessment, 72h notification clock (see `eu-ai-compliance`).
7. Write it down (date, key, window, impact, fix).

Safety nets that exist: GitHub secret scanning partner program scans **public** repos and forwards matches to providers — **Anthropic** (since Aug 2024), OpenAI, AWS, Stripe, Slack and others revoke or notify. It does not cover private repos unless you enable secret scanning/push protection there, and it doesn't help for keys pasted elsewhere.

## 5. Detection: stop leaks before they happen

- **Pre-commit:** `gitleaks protect --staged` or `detect-secrets` / `trufflehog` hook. Block commits containing key patterns.
- **Push protection:** enable GitHub secret scanning + push protection on every repo (free on public, available on private with GitHub Secret Protection).
- **History scan before publishing a repo or flipping it public:** `gitleaks detect` / `trufflehog git file://. --only-verified` over **all commits**, not just HEAD — a key deleted in a later commit is still in history. Also compare against the real secret values you hold (exact-match scan), since custom tokens have no pattern.
- **CI:** run the scanner on PRs; fail on findings.
- **Logs:** redact `Authorization`, `x-api-key`, `token`, `password`, query strings with `key=`; never log full request headers.

## 6. Special cases

- **LLM provider keys (Anthropic/OpenAI/ElevenLabs etc.):** one key per project + environment; set spend limits/alerts per key or workspace; prefer workspace/project-scoped keys; never ship keys to browsers or mobile apps — proxy through your backend.
- **n8n:** credentials are encrypted with `N8N_ENCRYPTION_KEY`. Losing it = all credentials unreadable; leaking it + DB = all credentials exposed. Back it up separately from the DB, set it explicitly (don't rely on the auto-generated file), and treat rotation as a migration (decrypt/re-encrypt — not a casual change). Workflow exports reference credentials by id/name only — verify before sharing JSON.
- **Webhook HMAC secrets:** verify signatures with constant-time compare; support two active secrets during rotation; include a timestamp in the signed payload to block replays.
- **OAuth refresh tokens:** stored like passwords; revoke on user offboarding; watch for silent expiry (a scheduled workflow that "just stops" is often an expired token — alert on "no new data", not only on errors).
- **Telegram bot tokens:** `/revoke` via @BotFather issues a new one instantly; session strings for user accounts (Telethon) are full account access — treat as the account password.
- **DB passwords:** app users least-privilege; separate read-only user for reporting/agents; prefer IAM auth or Vault dynamic creds.
- **SSH keys:** per-device keys with passphrases; remove from `authorized_keys` on device loss; pin SSH firewall rules to VPN/Tailscale ranges.
- **KMS / encryption keys:** envelope encryption; never store the key next to the data it encrypts; enable automatic key rotation (AWS KMS CMK yearly or custom period).

## 7. Agents and secrets

- **Never put a secret in a prompt, system prompt, tool description or memory.** Prompts get logged, traced, cached, and leak (OWASP LLM07).
- Tools hold credentials; the model only gets the tool. The model should never see the token value or be able to print it.
- **One credential per agent/tool, least privilege**, so you can revoke one agent without breaking others and audit logs show which agent did what.
- Coding agents: keep `.env` out of their read scope where possible; review generated code for pasted keys; run the pre-commit scanner on agent commits too.
- Don't pass secrets as CLI arguments (visible in `ps`, shell history) — use env or stdin.

## 8. Offboarding / handover checklist

- [ ] List every secret the person or vendor had access to (manager audit log, CI, shared vaults, servers).
- [ ] Rotate shared secrets; revoke their personal tokens, SSH keys, OAuth grants.
- [ ] Transfer ownership of keys tied to their personal accounts (API keys created under a personal login die with the account).
- [ ] Record what was rotated and when.

## Sources
- OWASP Secrets Management Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- Anthropic joins GitHub secret scanning partner program — https://github.blog/changelog/2024-08-20-anthropic-is-now-a-github-secret-scanning-partner/
- GitLab automatic response to leaked secrets — https://docs.gitlab.com/user/application_security/secret_detection/automatic_response
