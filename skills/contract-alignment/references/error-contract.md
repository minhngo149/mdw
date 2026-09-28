# Error Contract

Compare the error signals a producer emits with how each consumer interprets them. The question
is narrow: **does the consumer read the producer's error the way the producer means it?** Where
the error goes afterwards (retry layering, timeouts, dead letters, compensation) belongs to
`distributed-flow`.

## Contents

1. Extract the producer's error surface
2. Extract the consumer's error handling
3. Build the error matrix
4. Mismatch patterns
5. Asynchronous and webhook error contracts
6. Structural vs semantic mismatch
7. Evidence to collect
8. Common false positives

## 1. Extract the producer's error surface

Find every error the producer can return **across this boundary**, from the code that produces
the response, not from documentation:

- the handler's explicit error writes: status code, error code, body type
- the central error mapping: error middleware, exception handlers, `@ControllerAdvice`, an
  `errors.Is` switch, gRPC status mapping (`status.Error(codes.X, ...)`), a GraphQL error
  formatter. Record the mapping as a table
- framework-generated errors that reach the client: router `404`/`405`, body-size `413`,
  content-type `415`, validation-library `400`/`422`, panic-recovery `500`. Label them
  `INFERRED` unless the repository configures them
- the error body shape (`{code, message}`, RFC 7807 `{type, title, status, detail}`, gRPC
  `details`, GraphQL `errors[].extensions.code`) and any retry hint: a `retryable` flag,
  `Retry-After`, a gRPC status such as `UNAVAILABLE` vs `INVALID_ARGUMENT`

## 2. Extract the consumer's error handling

Read the client code between the call and the point where the result is used:

- which status codes it branches on, and whether it checks exact values (`== 409`) or ranges
  (`>= 500`)
- which error body it decodes, and which field it switches on (`code`, `type`, `error`)
- how it classifies each branch: success, idempotent success ("already done"), retryable,
  permanent failure, user-facing error, or crash
- its default branch: what an unlisted status or code becomes
- whether it reads the producer's retry hint at all

## 3. Build the error matrix

One row per producer error, plus one row per consumer branch that no producer error reaches:

```text
| Producer returns             | When                     | Consumer branch           | Consumer treats as  | Status     | Label     |
|------------------------------|--------------------------|---------------------------|---------------------|------------|-----------|
| 422 SEAT_TAKEN               | seat already reserved    | default: !=2xx            | permanent, generic  | PARTIALLY  | CONFIRMED |
| 429 + Retry-After            | rate limit               | 4xx -> permanent          | permanent           | MISALIGNED | CONFIRMED |
| (none)                       | -                        | 409 SEAT_CONFLICT -> retry| retryable           | dead branch| CONFIRMED |
```

The table is an illustration of the format, not a template to fill with these values.

## 4. Mismatch patterns

| Pattern | What it looks like | Typical consequence |
|---|---|---|
| Status or code mismatch | producer sends `422 SEAT_TAKEN`; consumer checks `409 SEAT_CONFLICT` | the intended handling never runs; the error takes the default branch |
| Retryable read as permanent | producer marks the error retryable (`503`, `UNAVAILABLE`, `retryable: true`, `Retry-After`); consumer fails permanently | a transient outage becomes a permanent business failure |
| Permanent read as retryable | producer sends a validation or business rejection; consumer retries on every non-2xx or with no limit | repeated calls that cannot succeed; duplicate side effects if the producer is not idempotent |
| Idempotent success read as failure | producer signals "already done" (`200` with the existing resource, or a dedicated code); consumer treats it as an error | a completed operation is marked failed on the consumer side |
| Failure read as success | producer returns `200` with an error body, or GraphQL `200` with `errors`; consumer checks only the status | the consumer proceeds with missing data |
| Body shape mismatch | producer returns RFC 7807 `detail`; consumer decodes `{code, message}` | the consumer's error code is always empty |
| Hint ignored | producer sends `Retry-After` or a `retryable` flag; consumer never reads it | the consumer's retry decision does not follow the producer's intent |

Report a retry-classification mismatch only with code evidence on both sides: where the
producer marks the error, and where the consumer classifies it. Retry count, backoff, and layering
belong to `distributed-flow`.

## 5. Asynchronous and webhook error contracts

Asynchronous boundaries have an error contract too, carried by responses or acks instead of error
bodies:

- **Webhooks.** The sender's retry policy (retries on non-2xx? on timeout? how long?) against the
  receiver's responses. A receiver that returns `500` for a permanently invalid payload is
  retried until the sender gives up; a receiver that returns `200` before processing, then fails,
  is never retried.
- **Request-reply over a broker.** Compare the reply's error fields or headers with what the
  requester reads.
- **Error events.** A producer that publishes `*.failed` events: compare their error codes and
  reasons with the consumer's branches, as in section 3.

When the other side is outside the repository, its half of the contract is `UNKNOWN`.

## 6. Structural vs semantic mismatch

| Kind | Example |
|---|---|
| Structural | the status, code string, or body field the consumer checks is not what the producer sends |
| Semantic | the values match but mean different things: producer `404` means "not yet created" (retry later) and consumer `404` handling means "never existed" (give up); producer `403` means the account is suspended and the consumer treats every `403` as an expired token to refresh |

## 7. Evidence to collect

```text
Producer   explicit error writes; central error mapping; body type; retry hints; framework errors that can reach the client
Consumer   status and code checks; error body decode type; classification of each branch; default branch; retry decision point
Tests      producer handler tests asserting error responses; consumer tests with mocked error responses (do the mocks match the producer?)
```

## 8. Common false positives

- **Range handling that is correct.** A consumer that retries every `5xx` and fails every `4xx`
  is aligned with a producer that uses `5xx` only for transient errors. Check the producer's
  actual use before reporting.
- **Unreachable producer errors.** An error code declared in a constants file but never returned
  on a path to this boundary cannot reach the consumer.
- **Wrapped clients.** A shared HTTP client or SDK may translate status codes into typed errors
  before the calling code sees them. Compare the producer's response with the wrapper's input,
  and the wrapper's output with the calling code.
- **Documented but unimplemented errors.** An OpenAPI response the handler never writes is
  documentation drift, not a consumer mismatch.
