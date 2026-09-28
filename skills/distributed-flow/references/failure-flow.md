# Failure Flow

Most of a flow's real behavior is in what happens when something goes wrong. The most valuable
findings are usually the state a failure leaves behind, not the error message it returns.

## Method

1. Take the hop list from the sync and async analysis.
2. For each hop, list the failure modes that hop can actually have (table below). Skip generic
   failures that no hop in this flow can produce.
3. For each failure mode, find:
   - **Detection:** the error check, catch block, status-code check, or timeout.
   - **Propagation:** return, rethrow, wrap, or map to a different error.
   - **Handling:** translated into a response, retried, logged, swallowed, or dead-lettered.
   - **State:** what has already been committed, published, or sent at that point.
4. Write each failure as a chain, and label every step:

```text
Trigger -> Detection -> System behavior -> State change -> Response / recovery
```

## Failure modes by hop type

| Hop | Failure modes to check |
|---|---|
| Inbound request | malformed input, validation, authentication, authorization, rate limit, payload too large, duplicate request |
| `IN_PROCESS` | business-rule violation, panic or unhandled exception |
| `DATABASE` | connection or pool exhaustion, constraint violation (unique, foreign key, check), deadlock or serialization failure, not found, timeout, commit failure |
| `REDIS` / cache | unavailable, timeout, key missing or expired. Is the flow fail-open (continues without the cache) or fail-closed (errors out)? |
| `HTTP` / `REST` / `GRPC` / `GRAPHQL` / external | connection refused, timeout, 4xx, 5xx, malformed response, and ambiguous success (the remote side acted but the response was lost) |
| Publish | broker unavailable, channel or connection closed, unroutable message, message not confirmed |
| Consume | undecodable (poison) message, handler error, crash mid-processing, redelivery, duplicate, out of order |
| Scheduled job | overlapping runs, partial batch failure, missed schedule |

## Where errors go

- **Error mapping.** Find the place that turns errors into responses: error middleware,
  exception handlers, `@ControllerAdvice`, an `errors.Is` switch, gRPC status mapping, a GraphQL
  error formatter. Record the mapping as a table (error to status or code). An unmapped error
  falls through to the framework default, often a 500 (`INFERRED`).
- **Swallowed errors.** Log-and-continue, empty `catch`, `_ =`, ignored return values,
  `.catch(() => {})`. These are critical findings: the caller sees success while the work is
  incomplete. Quote the evidence.
- **Errors on the error path.** A failure to record the failure (for example, updating a status
  to `FAILED` whose own error is only logged) leaves the state unchanged. Say what it stays as.
- **Crashes.** Recovery middleware turns a panic into a 500. In a worker, a crash usually means
  a restart and redelivery of unacked messages (`INFERRED` from the ack mode).

## Cross-cutting behaviors

**Retry.** Record where the retry lives, the attempt count, the backoff, which errors trigger
it, and whether the retried operation is idempotent. Check every layer: a loop in code, the
client library, a framework annotation, the job framework, broker redelivery, and infrastructure
(proxy or mesh). "No retry" is `CONFIRMED` only for the layers you inspected; the layers
outside the repository stay `UNKNOWN`. Layers multiply: 3 application retries times 3 SDK
retries is 9 attempts.

**Timeout.** A timeout is an ambiguous outcome, because the remote side may have completed.
Check what the code assumes, for example marking the operation failed while the remote charge
may have succeeded. Where timeouts are configured: `sync-flow.md`.

**Dead letter.** Record where dead-lettered messages go, who consumes them, and how they are
replayed. If the consumer or replay mechanism is not in the repository, it is `UNKNOWN`. Check
whether the dead-letter target is declared anywhere in the repository.

**Rollback.** Record which failures roll back which writes, and what cannot be rolled back:
remote calls, published messages, sent emails (see `transaction-flow.md`).

**Compensation and fallback.** Compensation undoes a completed step (refund, release, cancel).
A fallback substitutes a result (cached value, default, degraded response). Circuit breakers
(resilience4j, Polly, gobreaker, opossum) are a form of fallback; record their thresholds only
if they are configured in the repository.

**Duplicate request.** Look for an idempotency key: where it is stored, its scope and TTL, when
it is written relative to the work, and whether it is released or kept when the work fails. A
key that is kept after a failure blocks legitimate client retries. A key written after the work
does not stop concurrent duplicates. Also look for unique constraints and natural keys.

**Duplicate event.** Consumer idempotency (see `async-flow.md`).

**Concurrency conflict.** Two executions of the flow on the same entity at the same time. Look
for locks, version checks, unique constraints, and guarded updates (see `transaction-flow.md`).
If there are none, describe the concrete interleaving that breaks the flow, labeled `INFERRED`.

## Partial failure: check every gap

For every pair of consecutive side effects on **different** systems, ask: what if the process
dies, or the second call fails, right here? Document the state left behind and whether anything
repairs it. This is where the "stuck in `PENDING`", "charged but not recorded", and "saved but
never announced" findings come from.

## Format

Write one detailed chain for each important failure:

```text
FAILURE: PSP timeout during payment
  Trigger    POST {PSP_URL}/payments exceeds the 8 s client timeout         CONFIRMED  internal/booking/psp.go NewPSPClient()
  Detection  Pay() returns a wrapped deadline-exceeded error                CONFIRMED  internal/booking/psp.go Pay()
  Behavior   Confirm() marks the booking FAILED and releases the seats      CONFIRMED  internal/booking/service.go Confirm()
  State      bookings.status = FAILED, seats released; the PSP payment
             may still have succeeded                                       INFERRED   timeout is an ambiguous outcome
  Response   HTTP 502 {"error":"payment_unavailable"}                        CONFIRMED  internal/http/errors.go statusFor()
  Recovery   no PSP reconciliation found in repository                      UNKNOWN    resolve by asking the payments team
```

Summarize all failures in one table in the Markdown artifact:

```text
| Failure | Trigger | Detection | Behavior | State after | Response / recovery | Label |
```

Do not describe the external system's internal failure handling. Only record what this
repository observes and does about it.
