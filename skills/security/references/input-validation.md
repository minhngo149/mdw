# Input Validation

Trace untrusted input from each entry point and record what validation, normalization, and encoding
exist. Extends step 7 of `SKILL.md`. The point is to know what an attacker can put into each sink;
whether that reaches a vulnerability is decided in `injection.md`, `authorization.md`, and
`file-security.md`.

## Contents

1. Where input enters
2. Validation vs authorization
3. What to check per input
4. What to report

## 1. Where input enters

Every entry point has several input channels. Enumerate them for each:

```text
query parameters   path parameters   headers   cookies   JSON body   form data
multipart / file uploads   GraphQL variables   gRPC message fields   webhook payload
CLI arguments   environment variables   message payload (broker)   URL fragments (client)
```

Do not stop at the obvious body. Headers (`Host`, `X-Forwarded-*`, `Content-Type`, `Range`),
cookies, and path segments are attacker-controlled and often reach sinks (host-header poisoning,
cache keys, file paths, log lines).

## 2. Validation vs authorization

Keep these separate; conflating them hides findings:

- **Syntactic validation**: is the input well-formed? type, length, range, format, allowed values,
  encoding. Answers "is this a valid-looking value?"
- **Business authorization**: may *this caller* use this value? Answers "may they act on this
  resource / set this field?" (see `authorization.md`).

A value can pass validation and still be an authorization violation: a well-formed `order_id` that
belongs to another user, a valid `role` string the user may not assign. Validation never substitutes
for authorization, and authorization never substitutes for validation.

## 3. What to check per input

| Aspect | Question |
|---|---|
| Presence | required vs optional; what happens when absent (defaults that widen access?) |
| Type | is it coerced to the expected type, or used as a raw string? |
| Range / length | are there bounds? unbounded input enables DoS, overflow of downstream limits |
| Allowed values | is it constrained to a set (enum, allowlist)? crucial for identifiers reaching queries or paths |
| Normalization | is it canonicalized once, consistently? double-decoding and inconsistent normalization bypass filters |
| Encoding at output | is it encoded for the context it lands in (HTML, SQL identifier, shell, header, path)? |
| Server authority | are security-relevant values (price, role, status, ids) taken from the server, not the client? |

**Allowlist over denylist.** A denylist of "bad" characters or patterns is almost always
bypassable; prefer validating against a known-good set. Note where the code relies on a denylist.

**Validate at the boundary, once, in a form you control.** Validation scattered after branching, or
performed on a different representation than the sink uses (validate the decoded string, use the raw
one), is a common bypass.

## 4. What to report

Missing or wrong validation is usually reported as part of the vulnerability it enables (injection,
traversal, mass assignment), not on its own. Report validation directly when:

- An unbounded input reaches an expensive operation (large body, huge page size, deep nesting) →
  resource-exhaustion risk (`api-security.md`).
- A denylist filter is the only defense on a sink → note it as bypassable (`LOW`/`MEDIUM` depending
  on the sink).
- Security-relevant values are taken from the client where the server should be authoritative →
  cross-reference the authorization finding.

Do not report "no validation library" or "missing validation" abstractly. Tie it to a sink and an
impact, or record it as an `OBSERVATION`.
