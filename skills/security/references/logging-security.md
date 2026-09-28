# Logging Security

Whether sensitive values reach logs and error serialization. Extends step 11 of `SKILL.md`. What
gets returned to clients (vs logged) is in `data-protection.md`.

## Contents

1. Find the log sinks
2. What must not be logged
3. Common leak patterns
4. What to report

## 1. Find the log sinks

```text
fmt.Printf / println / console.log / print
log.Printf / logger.Info / logger.Debug / logger.Error / slog / zap / logrus / winston / pino
structured logging fields   request/response logging middleware   access logs
error serialization (logging an error object that embeds sensitive context)
panic/exception handlers that dump state
```

Debug-level logging is a frequent offender: it is often verbose, enabled in non-prod, and
occasionally left on. Note the log level and whether it is configurable.

## 2. What must not be logged

```text
passwords (plain or hashed)   full request bodies of auth endpoints
Authorization headers / bearer tokens   Cookie headers / session ids
API keys and secrets   password-reset / verification tokens   payment/card data
full PII where the log's audience is broader than the data's audience
```

## 3. Common leak patterns

| Pattern | Problem |
|---|---|
| Logging the whole request body on login/register/reset | captures the plaintext password |
| Logging the `Authorization` header on auth failure | captures bearer tokens; even "only on failure" logs valid-but-expired or mistyped-adjacent tokens — `LOW` but real. Logging it on success is worse. |
| Logging a reset/verification token "for debugging" | anyone with log access performs account takeover |
| `log.Printf("user: %+v", user)` | struct dumps include `password_hash`, tokens, and internal fields |
| Logging full external request/response for a payment/identity call | captures secrets and PII |
| Error objects that wrap the query or credentials | leaks via the error string |

Assess the **audience and retention** of the logs: a value in a log aggregator many people can read,
kept for months, is a broader exposure than a local stderr line. The repo rarely shows this, so the
runtime severity is often `INFERRED`; state the assumption.

## 4. What to report

| Finding | Typical severity |
|---|---|
| Password or secret written to logs | `MEDIUM`/`HIGH` (higher when the log audience is broad) |
| Auth token / reset token logged | `MEDIUM`/`HIGH`; a reset token logged is close to an account-takeover primitive |
| Full auth-endpoint request body logged | `MEDIUM` |
| Authorization header logged only on failure | `LOW` |
| Verbose struct/PII logging | `LOW`/`MEDIUM` by data class |

Do not reproduce the sensitive value in the finding. Cite the log call and what it would emit:
`internal/user/handler.go Handler.Login logs the raw request body, which contains the plaintext
password`. Fix: redact or omit the field, log identifiers not secrets, and lower the log level for
sensitive paths.
