# Secrets

Find, classify, and **redact** secrets in code and configuration. Extends step 10 of `SKILL.md`. The
first rule: the artifacts never contain a real secret value (`security-evidence.md` section 6).

## Contents

1. Where to look
2. What counts as a secret
3. Real vs placeholder
4. Hardcoded vs sourced
5. Sensitive environment variables
6. What to report

## 1. Where to look

```text
source code           config files (yaml/json/toml/ini/properties)
.env / .env.example   Dockerfiles and docker-compose
CI/CD (GitHub Actions, GitLab CI, Jenkinsfile, CircleCI)
Kubernetes / Helm / Terraform manifests
test fixtures and seed data       scripts       committed .pem/.key/.p12/.jks files
git history (a value removed from HEAD may still be in an earlier commit)
```

Useful searches (read matches; never paste values into output):

```bash
rg -n --no-messages -i 'secret|password|passwd|api[_-]?key|token|private[_-]?key|credential'
rg -n --no-messages 'AKIA[0-9A-Z]{16}|sk_live_|sk_test_|xox[baprs]-|-----BEGIN [A-Z ]*PRIVATE KEY-----|eyJ[A-Za-z0-9_-]{10,}\.'
rg -n --no-messages -i 'getenv|os\.environ|process\.env|System\.getenv|ENV\['
```

## 2. What counts as a secret

API keys, passwords, private keys, JWT/signing secrets, database credentials, cloud credentials
(AWS/GCP/Azure), OAuth client secrets, webhook secrets, encryption keys, session keys, access and
refresh tokens. Vendor prefixes help identify and classify without exposing the value: `AKIA…`
(AWS access key id), `sk_live_`/`sk_test_` (Stripe), `xoxb-`/`xoxp-` (Slack), `ghp_`/`gho_`
(GitHub), `-----BEGIN … PRIVATE KEY-----` (PEM), `eyJ…` (a JWT).

## 3. Real vs placeholder

Distinguish carefully — flagging placeholders as leaks destroys trust in the report:

| Likely real | Likely not |
|---|---|
| A live-mode prefix (`sk_live_`, `AKIA…`) with a full random body in committed code | `change-me`, `xxx`, `your-key-here`, `<REDACTED>`, empty string |
| A high-entropy string assigned to a secret name in non-example code | Values only in `.env.example`, `*.sample`, docs, tests clearly using dummy data |
| A private key file committed to the repo | A `test-secret` in a unit test, a fixture password |
| A default fallback in real code that is a plausible secret | A placeholder that could not authenticate anything |

A **default fallback** is the subtle case: `getenv("X", "some-value")`. If `some-value` is a real
credential or a signing secret that makes tokens forgeable, it is a finding even though it is
"only" a default (see below). If it is obviously a placeholder, it is not.

## 4. Hardcoded vs sourced

For each secret usage, determine how it is provided:

| Source | Assessment |
|---|---|
| Hardcoded literal in code | finding: committed secret. Compromised regardless of later removal. |
| Default fallback when env is unset | finding when the default is a real/functional secret: exposure is `INFERRED` (applies only when the env var is missing — check whether deployment sets it). A defaulted **signing** secret is worse: it makes tokens forgeable and can grant any role. |
| Loaded from environment | acceptable; check the value is not then logged or returned (`logging-security.md`, `data-protection.md`). |
| Loaded from a secret manager (Vault, AWS/GCP/Azure secrets, sealed secrets) | good practice; note it. |
| Committed key file | finding. |

Also check that a secret, once loaded, is not: logged, returned in an API response, put in a URL or
error message, or sent to a third party.

## 5. Sensitive environment variables

List the security-relevant env vars the app reads and how each is handled:

```text
DATABASE_URL   JWT_SECRET   AWS_SECRET_ACCESS_KEY   STRIPE_SECRET_KEY
PRIVATE_KEY   ENCRYPTION_KEY   *_WEBHOOK_SECRET   SESSION_SECRET   SMTP_PASSWORD
```

For each: is it hardcoded, defaulted, loaded from env, loaded from a manager, logged, or returned?
A missing secret with **no default and no startup check** is its own risk: the feature it protects
(HMAC verification, encryption) may run with an empty key. Note when a security control's secret can
be empty at runtime.

## 6. What to report

```text
Location   file + symbol/key
Type       what kind of credential (classified by prefix/name, not by value)
Real?      real | default fallback | placeholder | test fixture
Value      [REDACTED]   (always)
Impact     what the secret unlocks; for signing secrets, note token forgery / role escalation
Fix        rotate first (a committed secret is compromised), then remove from code, then
           source from env/secret manager, then add secret scanning to CI
Label      CONFIRMED that it is in the repo; exposure may be INFERRED/UNKNOWN
```

Severity: a live third-party credential in code is `HIGH` (often `CRITICAL` if it unlocks
production data or money). A defaulted signing secret is `HIGH — static risk` with `INFERRED`
exposure. A placeholder is not a finding. Never reproduce the value in the finding, the verification
step, or the reply to the user.
