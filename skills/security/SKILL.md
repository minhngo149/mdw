---
name: security
description: Use when reviewing a repository for security risks in authorized code review, hardening, threat modeling, or pre-release audit - analyzing authentication, authorization, IDOR/BOLA, multi-tenant isolation, injection (SQL, command, SSRF, path traversal), file uploads, session and token handling, secrets, cryptography, sensitive-data exposure, dependency and configuration risk, business-logic flaws, and race conditions, from source code, config, schema, and infrastructure definitions.
---

# Security

Analyze a real repository and identify security risks that are actually supported by its source
code, configuration, dependencies, schema, and infrastructure definitions. Record the result as
reusable text: a Markdown artifact (source of truth) and an HTML rendering of the same knowledge.

This is a **repository security analysis** for authorized defensive work: hardening, code review,
threat modeling, and pre-release audit. It reads code and reports risk with evidence. It does not
attack running systems.

**Core principle:** source code decides what the system does. A finding is a claim about code,
config, schema, or infrastructure that you have read. Documentation, comments, names, and
conventions are leads to verify, never findings on their own.

```text
Source Code           >  Documentation  >  Assumption
Evidence              >  Speculation
Actual Data Flow      >  Naming
Actual Authorization  >  Intended Authorization
Actual Configuration  >  Default Assumptions
Attack Path           >  Checklist
Impact + Exploitability + Evidence  >  Number of Findings
```

Every finding answers: **what can an attacker control, where does untrusted data enter, how does
it flow, what boundary does it cross, what protection exists, where is protection missing or
bypassable, what evidence proves it, and how can it be verified safely?**

## When to Use

- "Review this service for security issues before release." / "Audit the auth in this repo."
- "Is `GET /orders/:id` vulnerable to IDOR?" / "Can a user set their own role here?"
- "Check this repo for injection / SSRF / path traversal / hardcoded secrets."
- Threat modeling one component, hardening review, or incident follow-up on a code path.

Not for: attacking systems you are not authorized to test, penetration testing of live
infrastructure, writing exploits or malware, evasion tooling, or generating a generic security
checklist unmoored from this repository. If asked which flow to review and several exist, list the
entry points you found and ask.

## Evidence Labels and Confidence

Every finding carries a **Confidence** label. These definitions hold for this skill, its
references, and both artifacts. They are the same three labels the other MDW skills use.

| Label | Use when | Must include |
|---|---|---|
| `CONFIRMED` | You read the code, config, schema, or in-repo infrastructure that creates the risk, on a path that is reachable | file + symbol; line range only if you read those lines |
| `INFERRED` | Strongly implied by code you read but not directly proven: framework defaults, a runtime-selected wiring, the consequence of a confirmed fact, exposure that depends on deployment | the evidence + the reasoning step |
| `UNKNOWN` | The repository does not contain enough to decide: runtime exposure, deploy-time secrets, external systems, broker or WAF policy, code in another repository | what would resolve it |

- Documentation, comments, and names alone never make a finding `CONFIRMED`. When they disagree
  with code, code wins and the disagreement is recorded as **documentation drift** (for example,
  a README that claims ownership is enforced when the handler has no check).
- Never upgrade `INFERRED` to `CONFIRMED` without reading the code. Never fill an `UNKNOWN` with a
  "typical" value.
- An **absence** is `CONFIRMED` only for the places you inspected: "no ownership check in
  `OrderHandler.Get` or its repository call" is `CONFIRMED`; a check added by a gateway or another
  repository remains `UNKNOWN`.
- **Exploitability is separate from existence.** You can `CONFIRMED` that a check is missing while
  the runtime *impact* stays `INFERRED` or `UNKNOWN` because it depends on deployment. Say both.

## Severity

Use `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `OBSERVATION`. Base severity on exploitability, impact,
exposure, required privileges, attack complexity, and affected assets — not on the fact that a
checklist item exists.

For static analysis, where runtime exposure and real traffic are unknown, qualify the severity:
write `HIGH — static risk`, not a claim that the issue has been exploited or measured. Use
`potential`, `static risk`, and `requires verification` where they fit. Details:
`references/security-evidence.md`.

## Workflow

Copy this checklist and complete it in order. **Adapt it to the technologies actually present** —
do not run every category against every repository. A CLI with no HTTP surface has no CORS
analysis; a repo with no broker has no message-security analysis. Skip with one line saying why.

```text
[ ] 1.  Scope: what is being reviewed, and the technologies present
[ ] 2.  Attack surface: entry points where untrusted data enters
[ ] 3.  Trust boundaries: where data crosses a privilege or system boundary
[ ] 4.  Threat model: assets, actors, sensitive operations, attack paths
[ ] 5.  Authentication analysis
[ ] 6.  Authorization analysis (IDOR/BOLA, privilege escalation, tenant isolation)
[ ] 7.  Input / output analysis
[ ] 8.  Injection analysis (SQL, command, SSRF, path traversal, template, XSS)
[ ] 9.  Session / token analysis
[ ] 10. Secret analysis
[ ] 11. Data protection analysis (exposure, logging, error handling, crypto)
[ ] 12. File / resource access analysis
[ ] 13. External integration analysis (webhooks, brokers, third-party APIs)
[ ] 14. Dependency / configuration / container analysis
[ ] 15. Business-logic and race-condition analysis
[ ] 16. Threat-path reconstruction: chain findings into attack paths
[ ] 17. Finding validation: prune false positives, set severity and confidence
[ ] 18. Write the Markdown artifact, then the HTML artifact
[ ] 19. Self-check
```

### 1. Scope and technology discovery

Identify only what the review needs: language, framework, application type (REST / GraphQL / gRPC
/ web app / CLI / worker / cron / webhook / event consumer), database, cache, message broker,
authentication mechanism, authorization model, external integrations, file storage, cloud
services, deployment, CI/CD, and containers. Do not assume the repository is an HTTP API. Start
from manifests, entry points, and deployment files; follow dependencies outward. Techniques:
`references/threat-model.md`.

### 2. Attack surface

Enumerate externally reachable or security-sensitive entry points: HTTP/REST routes, GraphQL
resolvers, gRPC methods, webhooks, message consumers, CLI arguments, environment variables, file
uploads, admin endpoints, internal APIs, scheduled jobs. For each meaningful entry point record:
entry, authentication, authorization, input, sensitive operations, external dependencies, data
access. `references/api-security.md`, `references/threat-model.md`.

### 3. Trust boundaries

Identify each transition where data crosses a privilege or system boundary (Internet → API →
application → database; client → gateway → service → service → database; and every external API,
webhook, broker, file store, OAuth provider, payment provider, admin interface). For each, ask:
*does the application validate and authorize data crossing this boundary?* `references/threat-model.md`.

### 4. Threat model

Build a lightweight threat model **from the actual repository**: assets, actors (anonymous,
authenticated user, admin, service account, internal service, third-party, malicious external
system), trust boundaries, entry points, sensitive operations, sensitive data, attack paths,
security controls. Do not assume an actor has privileges the repository does not grant.
`references/threat-model.md`.

### 5. Authentication

Determine how identity is established and validate the mechanism against its own rules: JWT,
session cookies, OAuth/OIDC, API keys, Basic Auth, mTLS, service tokens, signed webhooks, custom.
For tokens, inspect signature verification, algorithm handling, expiration, issuer, audience,
subject, refresh handling, and cookie flags. Claim a missing check only when source code supports
it. `references/authentication.md`.

### 6. Authorization

**Primary focus.** For each sensitive operation: *who* can perform it, on *what* resource, with
*which* operation, and *where* is that enforced? Look for RBAC/ABAC, ownership checks,
resource-level authorization, role and permission checks, middleware, policy engines. Detect
IDOR/BOLA, vertical and horizontal privilege escalation, missing authorization, and mass
assignment of privileged fields. When the repo appears multi-tenant, check that tenant context is
applied to every read, write, cache key, job, file path, and outbound call — a field named
`tenant_id` does not prove isolation. `references/authorization.md`.

### 7. Input / output

Trace untrusted input from each entry point: query and path parameters, headers, cookies, JSON,
form data, GraphQL/gRPC input, webhook payloads, uploaded files, CLI args, env vars. Distinguish
**syntactic validation** (type, length, allowed values) from **business authorization**. Record
what validation, normalization, and encoding exist and where. `references/input-validation.md`.

### 8. Injection

Analyze **actual data flows**, not string proximity. For each sink, decide whether a safe API or
parameterization is used, and whether an attacker controls the dangerous part: SQL and NoSQL,
command/OS, LDAP, template/expression, GraphQL, header/CRLF, HTML/XSS, SSRF, path traversal.
Parameterized *values* do not make *identifiers* (table, column, `ORDER BY`) safe — analyze those
separately. `references/injection.md`, `references/file-security.md`.

### 9. Session / token

For sessions: cookie `Secure`, `HttpOnly`, `SameSite`, expiration, rotation, logout invalidation,
fixation, CSRF. For tokens (access, refresh, reset, verification, API, webhook signature): entropy
source, expiration, storage, rotation, revocation, single-use, audience, scope, signature
verification. Do not assume browser behavior from framework defaults unless config confirms it.
`references/session-security.md`, `references/authentication.md`.

### 10. Secrets

Search code and config for API keys, passwords, private keys, JWT secrets, DB credentials, cloud
credentials, OAuth secrets, webhook secrets, encryption keys, tokens. Distinguish a real secret
from a placeholder, example, test credential, or public identifier. **Never print a secret value
in the artifacts** — redact it and record only its location and type. `references/secrets.md`.

### 11. Data protection

Identify sensitive data and trace whether it reaches responses, logs, errors, metrics, events,
URLs, or headers. Check logging for secrets and PII, error handling for internal detail leakage
(stack traces, SQL, paths, hostnames) returned to clients, and cryptographic usage (password
hashing, encryption, signing, randomness) for concrete insecure implementations — not generic
preference. `references/data-protection.md`, `references/logging-security.md`,
`references/cryptography.md`.

### 12. File / resource access

For file open/read/write/delete, downloads, uploads, archive extraction, and template loading,
trace user-controlled paths and check normalization, allowlists, canonicalization, and storage
location. For uploads, check size limits, content-type and magic-byte validation, filename
handling, storage location, and execution permission. Do not trust `Content-Type`, extension, or
client filename. `references/file-security.md`.

### 13. External integrations

For webhooks: signature verification, timestamp/replay protection, secret validation, idempotency.
For brokers (RabbitMQ, Kafka, NATS): producer/consumer trust, message validation, authorization,
tenant context, replay, idempotency. For third-party APIs: how their responses are trusted. An
HTTPS webhook receiving JSON is not automatically secure. `references/api-security.md`,
`references/injection.md` (SSRF).

### 14. Dependency / configuration / container

Read dependency manifests and lockfiles and record versions. **Do not claim a CVE unless a tool or
evidence in this session confirms it, and never fabricate CVE identifiers**; otherwise write
"version identified; known-vulnerability status not verified". Inspect Dockerfiles, compose, k8s,
Terraform, Helm, CI/CD, and proxy/TLS config for debug mode, dev config in production, exposed
ports, privileged or root containers, disabled TLS, and secret handling. Do not assume deployment
files represent production unless clearly indicated. `references/dependency-security.md`,
`references/configuration-security.md`.

### 15. Business logic and race conditions

Analyze security-sensitive business rules present in the repo: double spending, duplicate
redemption, price/quantity manipulation, workflow/approval bypass, status manipulation, ownership
transfer, refund/coupon abuse, replay. For race conditions, identify the concrete concurrent path
(TOCTOU, check-then-act on balance/permission, duplicate requests, missing locks or unique
constraints). Do not claim a race without naming the interleaving. `references/authorization.md`,
`references/data-protection.md`.

### 16. Threat-path reconstruction

Chain findings into attack paths. The most valuable results are chains: mass assignment of `role`
+ role-trusting authorization = privilege escalation to admin; `ORDER BY` injection + error echo =
data extraction. Draw each as a plain-text attack path (see `references/security-evidence.md`).

### 17. Finding validation

Before writing, prune. For each candidate finding, confirm: untrusted input actually reaches the
sink, no safe API or control neutralizes it, the code is reachable, and the impact is real for the
actors this repo grants. Demote what does not survive to `OBSERVATION` or drop it. Set severity and
confidence. See the false-positive rules in `references/security-evidence.md`.

### 18. Write the artifacts

Read `references/security-evidence.md` first. It defines the finding model, severity and confidence
rules, the redaction rules, the Markdown section contents, the attack-path text format, and the
HTML skeleton.

**Output contract**

- Write exactly two files: `<scope>-security.md` and `<scope>-security.html`. Default location:
  `mdw/security/` at the analyzed repository's root; a location the user gives takes precedence. If
  the file exists, read it first and replace it only if it is an earlier MDW security artifact for
  the same scope.
- Markdown is the source of truth. Write it first, then derive the HTML from it. The HTML adds no
  findings and drops none.
- **Never write a real secret value into either artifact.** Redact to `[REDACTED]` or a masked form
  (`sk_live_****…****`) and record only location and type.
- Never produce Mermaid, `.mmd`, `.drawio`, `.png`, `.jpg`, `.svg`, inline `<svg>`, `<canvas>`,
  images, or any other diagram format. Draw attack paths as plain text in Markdown and with
  HTML/CSS in HTML.
- The HTML is one self-contained file with inline CSS. It makes no external requests (no CDN,
  fonts, or remote scripts), uses no framework, needs no build step, opens directly from disk, and
  is fully readable with JavaScript disabled.

**Markdown structure.** Keep every heading, in this order. When a section does not apply, write one
line saying so and why (for example, "None: no file access on the reviewed paths"). Never pad.

```md
# <Scope> Security Analysis

## 1. Scope
## 2. Application Attack Surface
## 3. Trust Boundaries
## 4. Authentication
## 5. Authorization
## 6. Input / Output Security
## 7. Injection Analysis
## 8. Session / Token Security
## 9. Secrets
## 10. Data Protection
## 11. File / Resource Security
## 12. External Integrations
## 13. Dependency / Configuration Security
## 14. Business Logic Security
## 15. Security Findings
## 16. Safe Verification Plan
## 17. Confirmed / Inferred / Unknown
## 18. Open Questions
## 19. Evidence
```

Then reply to the user with the two paths and a short summary: the scope, the top findings by
severity, any documentation drift, and the counts of `CONFIRMED` / `INFERRED` / `UNKNOWN`. Do not
paste the whole document into the reply.

### 19. Self-check

```text
[ ] Every finding cites a file and symbol you actually read; no line number you did not read
[ ] Every finding has Category, Severity, Location, Attack Surface, Evidence, Attack Path,
    Security Impact, Confidence, How to Verify, Recommended Fix, Trade-offs
[ ] Every finding names the untrusted input, the sink, and why no control neutralizes it
[ ] Missing-control findings say what was inspected (CONFIRMED absence) vs UNKNOWN
[ ] Severity for static findings is qualified ("HIGH — static risk"), not "exploited"
[ ] No secret value appears in either artifact; all are redacted
[ ] No CVE identifier is stated unless verified this session; unverified deps say so
[ ] No finding is just "uses HTTP / has env vars / uses JWT / has an admin endpoint"
[ ] Documentation contradicted by code is listed as drift
[ ] Safe verification is non-destructive and uses test accounts, never production
[ ] The report never says "the system is secure"; it says "no issue found in the analyzed path"
[ ] The HTML makes the same findings as the Markdown, in the same section order
[ ] Only .md and .html were written; no Mermaid, SVG, images, or diagram files
```

## Never Invent

Users, attackers, traffic, production exposure, credentials, permissions, security controls,
vulnerabilities, or CVE identifiers. When uncertain, write `UNKNOWN`. When implied but not
observed, write `INFERRED` with the reasoning. Never conclude "the system is secure" from the
absence of an obvious issue — write "no issue was identified in the analyzed path".

| Thought | Reality |
|---|---|
| "The README says ownership is enforced" | Docs are leads. Find the check in code; if it is missing, that is a finding **and** documentation drift. |
| "User input reaches a query, so it is SQL injection" | Only if the dangerous part is attacker-controlled and no parameterization/safe API neutralizes it. Trace the flow. |
| "It makes an outbound HTTP request, so it is SSRF" | Only if the attacker controls the URL/host/scheme and no allowlist or private-IP block stops it. |
| "`POST`/`PUT`/`DELETE` exist, so it needs CSRF protection" | CSRF depends on cookie-based auth in a browser. Bearer-token APIs are usually not in scope. Check the auth model. |
| "This dependency is old, so it has CVE-XXXX-YYYY" | Never fabricate CVEs. Record the version and "known-vulnerability status not verified" unless a tool confirmed it. |
| "`USER root` in the Dockerfile is CRITICAL" | Report it as a concrete risk with real impact and severity, not an automatic critical. |
| "There is a `tenant_id` column, so tenancy is isolated" | A field is not enforcement. Confirm it filters every read, write, cache key, and job. |
| "No rate limiter, so it is a vulnerability" | State "no rate-limiting control identified for this sensitive endpoint; evaluate against deployment and threat model." Not an automatic finding. |
| "The secret is right here in config.go" | Record location and type; redact the value. Never copy a credential into the report. |
| "I found 30 issues" | Impact + exploitability + evidence beats count. Prune false positives; chain the real ones into attack paths. |
| "This would be clearer in Mermaid" | Output contract: plain text in `.md`, HTML/CSS in `.html`. |

## References

Read each reference when its step needs it. They extend this workflow; they do not repeat it.

| File | Read when |
|---|---|
| `references/threat-model.md` | Steps 1-4: scope, attack surface, trust boundaries, actors, assets, attack paths |
| `references/authentication.md` | Step 5, and step 9 for tokens: identity mechanisms, JWT/session/OAuth validation |
| `references/authorization.md` | Step 6 and step 15: IDOR/BOLA, privilege escalation, mass assignment, tenant isolation, logic |
| `references/input-validation.md` | Step 7: tracing untrusted input, validation vs authorization |
| `references/injection.md` | Step 8 and step 13: SQL, command, SSRF, path traversal, template, header, XSS |
| `references/secrets.md` | Step 10: finding, classifying, and redacting secrets and sensitive env vars |
| `references/session-security.md` | Step 9: cookies, CSRF, CORS, session lifecycle |
| `references/api-security.md` | Steps 2 and 13: API controls, rate limiting, webhooks, broker/message security |
| `references/data-protection.md` | Step 11: sensitive-data exposure, response/error leakage |
| `references/dependency-security.md` | Step 14: manifests, lockfiles, the no-fabricated-CVE rule |
| `references/configuration-security.md` | Step 14: Docker, compose, k8s, Terraform, CI/CD, TLS, containers |
| `references/file-security.md` | Step 12: path traversal, upload validation, archive extraction |
| `references/cryptography.md` | Step 11: hashing, encryption, signing, randomness, password and token security |
| `references/logging-security.md` | Step 11: secrets and PII in logs and error serialization |
| `references/security-evidence.md` | Steps 16-18: finding model, severity/confidence, redaction, attack-path format, Markdown sections, HTML skeleton |
