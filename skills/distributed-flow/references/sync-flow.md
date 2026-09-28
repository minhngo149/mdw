# Synchronous Flow

A hop is `SYNC` when the caller blocks or awaits until it has the receiver's result, and what it
does next depends on that result. Synchronous hops form the request's critical path. Their
latencies add up, and each one's failure reaches the caller directly.

## What to capture per hop

| Field | Where to find it |
|---|---|
| Call site | file and symbol of the caller |
| Receiver | function, process, or system |
| Protocol | `IN_PROCESS`, `HTTP`, `REST`, `GRAPHQL`, `GRPC`, `DATABASE`, `REDIS`, `OTHER_EXTERNAL_API`, and so on |
| Request | HTTP method and path, RPC name, query, or command, plus the fields that matter |
| Response use | what the caller does with the result and with each kind of error |
| Timeout | client construction, context deadline, per-call option (table below) |
| Retry | client wrapper, interceptor, annotation, or manual loop (table below) |
| Context propagation | whether the inbound request's context, deadline, or cancellation is passed down |

Record in-process hops briefly, one line each with file and symbol, in the main flow. Hops that
leave the process also get a row in the boundary table (see `service-boundary.md`).

## Middleware and interceptors

Record the chain in execution order and mark which links can end the request early, for example
an auth check returning 401 before the handler runs. Where to look: router `Use()` and route
groups, Express `app.use`, Nest guards, pipes, and interceptors, Spring filters and
`HandlerInterceptor` and the Security filter chain, gRPC interceptors, the Django `MIDDLEWARE`
setting, the ASP.NET pipeline. A route group applies its middleware only to routes registered
inside that group, so check which group the entry point is in.

## Where timeouts hide

| Stack | Places to check | Defaults worth knowing (`INFERRED` when relied on) |
|---|---|---|
| Go | `http.Client{Timeout}`, `context.WithTimeout`/`WithDeadline`, `Transport` dial, TLS, and response-header timeouts, `http.Server` read and write timeouts, driver statement timeouts, gRPC deadline from `ctx` | A zero-value `http.Client` has no timeout |
| Node | axios `timeout`, fetch with `AbortSignal.timeout`, got and undici options, server `requestTimeout` | axios `timeout` defaults to 0 (none) |
| JVM | RestTemplate, WebClient, Feign, and OkHttp timeouts, resilience4j `TimeLimiter`, JDBC query timeout, `@Transactional(timeout)` | varies by client; check its construction |
| Python | requests `timeout=`, httpx `Timeout`, aiohttp `ClientTimeout`, gunicorn `timeout` | requests has no timeout by default; httpx defaults to 5 s |
| .NET | `HttpClient.Timeout`, Polly timeout policies | `HttpClient.Timeout` defaults to 100 s |

A client constructed without a timeout is a `CONFIRMED` fact. That it can hang indefinitely is
`INFERRED` from the library default. If a timeout value comes from config, cite both the code
that reads it and the in-repo value, and note any fallback used when the value is missing or
invalid.

## Where retries hide

- Client wrappers: go-retryablehttp, axios-retry, urllib3 `Retry`, tenacity, Polly
- Annotations and decorators: Spring `@Retryable`, resilience4j `@Retry`
- Built-in SDK retries (many cloud SDKs retry by default): `INFERRED` unless configured in code
- gRPC service config `retryPolicy`
- Manual loops around the call
- Service mesh or proxy (Istio, Envoy, Linkerd): only from manifests in the repository; otherwise `UNKNOWN`

For each retry, record where it is, how many attempts, the backoff, which errors trigger it, and
whether the retried operation is idempotent. Retry layers multiply.

## Fan-out and parallelism

Parallel calls (`errgroup`, `Promise.all`, `CompletableFuture.allOf`, `asyncio.gather`) are
still `SYNC` if the caller waits for all of them. Record which calls run in parallel and what
happens when only one fails: cancel the others, wait for all, or accept partial results.

## Hazards that need their own note

| Hazard | Why it matters |
|---|---|
| Network call inside an open database transaction | Holds the connection and locks for the call's duration. Rolling back does not undo the remote effect. |
| Remote side effect before the local commit | The remote effect can exist with no local record. |
| Local commit before the remote side effect | The local record can exist with no remote effect. |
| Timeout on a non-idempotent call | Ambiguous outcome: the remote side may have completed. Check what the caller assumes. |
| Response sent before the work finishes (streaming, `go func()` after writing the response) | That remaining work is actually `ASYNC`. Treat it with `async-flow.md`. |

## How to document

Numbered steps in the main flow, one hop per line:

```text
1. Client -> Booking API               REST SYNC      POST /bookings                      CONFIRMED  internal/http/routes.go Routes()
2. AuthMiddleware                      IN_PROCESS     rejects missing token with 401     CONFIRMED  internal/http/auth.go RequireToken()
3. BookingHandler.Create               IN_PROCESS     decode, validate, call service      CONFIRMED  internal/http/booking.go Create()
4. BookingService.Reserve -> PostgreSQL DATABASE SYNC tx T1: lock seats, insert booking   CONFIRMED  internal/booking/repo.go Reserve()
5. BookingService.Reserve -> PSP        REST SYNC      POST /payments, 8 s timeout        CONFIRMED  internal/booking/psp.go Pay()
```

Add a plain-text diagram (conventions in `text-flow-format.md`) when there is branching or more
than one process involved. Put timeout and retry facts in the boundary table columns.
