# Data Protection

Sensitive-data exposure and where it leaks. Extends step 11 of `SKILL.md`. Logging detail is in
`logging-security.md`, crypto in `cryptography.md`, secrets in `secrets.md`.

## Contents

1. Classify sensitive data
2. Exposure paths
3. Responses
4. Errors
5. What to report

## 1. Classify sensitive data

Not every field is sensitive the same way. Classify what the repo actually stores and handles:

| Class | Examples | Concern |
|---|---|---|
| Secrets | passwords, tokens, keys | never exposed anywhere; hash/encrypt at rest |
| Authn material | password hashes, reset tokens, session ids | never in responses or logs |
| Payment | card data, bank details | usually must not be stored raw; PCI scope |
| Personal (PII) | name, email, address, phone, government id | exposure to the wrong party; regulatory |
| Internal | internal ids, system config, feature flags | enables further attacks; lower direct impact |

Explain the actual exposure path rather than labeling every personal field "sensitive". A user's own
email in their own response is fine; another user's email in the response is an authorization
finding.

## 2. Exposure paths

Trace whether each sensitive value reaches a place it should not:

```text
API response body   logs   error messages   metrics/traces   analytics events
message payloads   database (unhashed/unencrypted)   cache   URLs / query strings
request/response headers   third-party calls   client-side storage
```

## 3. Responses

- **Over-serialization**: returning a whole model/entity when only some fields are meant to be
  exposed. Look for `password_hash`, `role`, internal flags, other users' fields slipping into
  responses via a struct/serializer that includes them. (In Go, a `json:"-"` tag hides a field —
  confirm sensitive fields have it; in other stacks look for explicit DTOs vs raw model returns.)
- **Verbose objects**: list endpoints returning full records including fields the caller may not see.
- **Cross-user exposure**: a response containing another user's data is an authorization finding
  (`authorization.md`).

## 4. Errors

Distinguish **internal logging** from **the external response**. Detailed errors are appropriate in
internal logs; returned to clients they leak implementation detail.

Check what the error path returns to the caller:

```text
stack traces   SQL text or DB driver errors   file-system paths   internal hostnames/IPs
library versions   tokens or credentials in an error   the raw error string from a lower layer
```

A common pattern: a handler returns `err.Error()` (or the framework's default 500 page in debug
mode) straight to the client. When the underlying error is a database or parse error, this leaks
schema and internals, and it strengthens injection findings (error-based extraction —
`injection.md`). Report the leak, and note the chain when it amplifies another finding.

Find the central error mapper (error middleware, exception handler, `@ControllerAdvice`, a status
switch). Confirm it maps internal errors to generic client messages and logs the detail server-side.
An unmapped error usually falls through to a framework default (`INFERRED`); check whether that
default is verbose (debug/dev mode — `configuration-security.md`).

## 5. What to report

| Finding | Typical severity |
|---|---|
| Password hash / token / secret in a response | `HIGH`/`CRITICAL` |
| Another user's PII in a response (authz) | `HIGH` — cross-reference authorization |
| Raw internal errors returned to clients | `LOW`, or `MEDIUM` when it enables injection extraction |
| Sensitive data in logs | see `logging-security.md` |
| PII stored/transmitted unencrypted where required | `MEDIUM`, note the requirement source |

For at-rest and in-transit protection (hashing, encryption, TLS), see `cryptography.md` and
`configuration-security.md`; report concrete insecure implementations, not generic "should encrypt"
advice unless a specific sensitive field is demonstrably unprotected.
