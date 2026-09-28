# API Security

API-level controls, rate limiting and abuse resistance, webhooks, and broker/message security.
Extends steps 2 and 13 of `SKILL.md`. Per-endpoint authentication and authorization are in
`authentication.md` and `authorization.md`; this file covers the controls around them.

## Contents

1. Per-endpoint controls
2. Rate limiting and abuse
3. Webhooks
4. Message / event security
5. Resource exhaustion

## 1. Per-endpoint controls

For each sensitive endpoint, record which controls exist and which are missing — but report a
missing control only with its risk context, never as an automatic finding:

```text
authentication   authorization   input validation   output encoding
rate limiting   request size limit   pagination limit   method restriction
content-type enforcement   error handling   CORS   CSRF (if cookie auth)
```

The finding is an endpoint that performs a **sensitive operation without an appropriate control**,
explained: which operation, which control, why it matters here.

## 2. Rate limiting and abuse

Look for a rate limiter and where it applies:

```text
IP-based   user-based   API-key-based   login-attempt limit   password-reset limit
OTP/verification limit   resource-creation limit   per-route limiter middleware
```

Endpoints where abuse resistance matters most: authentication, password reset, verification/OTP,
expensive operations, resource creation, and public unauthenticated APIs.

**If no rate limiter is found, do not automatically call it a vulnerability.** Write:

> No rate-limiting control was identified for this sensitive endpoint. Abuse resistance should be
> evaluated against the deployment (a gateway or WAF may enforce it — `UNKNOWN` from the repo) and
> the threat model.

Raise it to a real finding only when the endpoint's function makes unlimited attempts materially
dangerous (credential stuffing on login, reset-token guessing, cost amplification) **and** no
in-repo or clearly-required external control covers it. Note when a README claims rate limiting that
the code does not implement — that is documentation drift.

## 3. Webhooks

An inbound webhook is an unauthenticated entry point unless the code proves the sender. An HTTPS
endpoint receiving JSON is **not** secure by that fact. Check:

| Control | What goes wrong without it |
|---|---|
| Signature verification (HMAC over the raw body with a shared secret) | anyone can forge events |
| Secret present and non-empty | if the secret defaults to empty/unset, the HMAC is forgeable — check `secrets.md`; note when it can be empty at runtime |
| Raw-body signing | verifying a re-serialized body (not the exact bytes) can be bypassed |
| Constant-time comparison | a non-constant-time `==` on the signature is a `LOW` timing risk |
| Timestamp + replay protection | a captured valid event can be replayed unless the handler is idempotent or checks a timestamp/nonce |
| Idempotency | duplicate delivery (providers retry) must not double-apply effects |
| Payload validation and cross-checks | trusting event fields blindly; verify against the provider's API where the code does (a stronger design) |

Assess replay in light of idempotency: a replayed event that only re-runs an idempotent UPDATE with
a server-side amount check is low risk; a replayed event that credits an account is high risk.

## 4. Message / event security

For brokers (RabbitMQ, Kafka, NATS, SQS):

| Question | Why |
|---|---|
| Are consumed messages trusted because they are "internal"? | An attacker who reaches the broker, or a compromised producer, injects malicious messages. Internal ≠ authenticated. |
| Is the payload validated and authorized like external input? | Consumers often skip the validation their HTTP siblings do. |
| Is tenant/user context carried and re-checked, not assumed? | Cross-tenant effects from a message lacking scoping. |
| Replay and duplicate handling (idempotency)? | At-least-once delivery replays messages. |
| Sensitive data in payloads? | Payloads may be logged or persisted in the broker (`data-protection.md`). |

Report "internal messages are implicitly trusted" only when you can see the consumer acting on
unvalidated payload fields in a sensitive way.

## 5. Resource exhaustion

Availability findings worth noting when concrete:

```text
no request body size limit on an endpoint that reads the whole body
no pagination limit (client sets an unbounded page size)
unbounded GraphQL query depth/complexity, or batching without limits
no server read/write/idle timeouts (slowloris, connection exhaustion)
zip/archive extraction without size/count limits (zip bomb) — see file-security.md
regexes on user input that can backtrack catastrophically (ReDoS)
```

Rate these by exposure and impact; many are `LOW`/`MEDIUM` and depend on deployment.
