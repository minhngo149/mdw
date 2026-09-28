# API Contract

Compare what a server (the producer) accepts and returns with what each client (the consumer)
sends and reads, across HTTP, REST, GraphQL, and gRPC. Payload field semantics are covered in
`data-contract.md` and error responses in `error-contract.md`. This file covers pairing and the
request/response surface.

## Contents

1. Pair the client call with the server route
2. What to compare
3. GraphQL specifics
4. gRPC and protobuf specifics
5. Structural vs semantic mismatch
6. Evidence to collect
7. Common false positives

## 1. Pair the client call with the server route

A client and a route are paired only when the request the client builds reaches that route.
Rebuild the full request on the client side, then find the route that serves it:

```text
client call -> base URL (config key -> value -> which deployable listens there)
            -> path concatenation and path params -> method -> route registration -> handler -> decode type
```

- Resolve the base URL through config, environment, compose, or manifests to a deployable in the
  repository. If it points outside the repository (a third-party API, another team's service),
  the server side is `UNKNOWN`; compare against a vendored spec or generated client only as
  documentation-level evidence.
- Include what sits between them when the repository defines it: API gateway routes, ingress
  rewrites, reverse-proxy prefixes (`/api` stripped), versioned mounts (`/v1` added by a router
  group). Without that evidence, a prefix difference is `INFERRED` or `UNKNOWN`, not `CONFIRMED`.
- Generated clients (OpenAPI generators, gRPC stubs, GraphQL codegen) tie the client to a spec.
  Check that the spec the client was generated from is the one the server implements; a stale
  generated client is a common source of drift.

## 2. What to compare

| Element | Producer (server) | Consumer (client) |
|---|---|---|
| Method and path | registered method, path template, version prefix, trailing slash handling | method and path the client builds |
| Path params | names, formats (UUID, int, slug), which ID they mean | values the client puts there (public ID vs internal ID) |
| Query params | names, types, array encoding (`a=1&a=2` vs `a=1,2` vs `a[]=1`), defaults | names and encoding the client sends |
| Headers | required headers: content type, auth, tenant, idempotency key, API version, `Accept` | headers the client sets |
| Auth context | the identity the server derives (user token, service token, API key) and the claims it reads | the credential the client attaches. Record only whether they agree on which credential is sent; whether the check is sound is `security` |
| Request body | decode type, required fields, validation rules, defaults applied | fields and values the client serializes |
| Response status | success status codes the handler writes (`200`, `201`, `202`, `204`) | success codes the client accepts; a client that checks `== 200` rejects a `201` |
| Response body | encode type and fields | decode type and every field the client reads |
| Pagination, sorting, filtering | parameter names, allowed values, page base, page size cap, cursor field | parameters the client sends and how it walks pages |
| Content negotiation | `Content-Type` produced; compression; envelope (`{data: ...}`) | what the client parses; does it unwrap an envelope the server does not send? |

For each element, the server's registered behavior wins over its OpenAPI description when they
differ. Record the difference as documentation drift, then compare the client with the code.

## 3. GraphQL specifics

- Validate the client's operation against the server schema: every selected field exists on the
  type, arguments match names and types, and variables match input types.
- Nullability: the schema's `String!` vs `String`. A resolver that can return `null` for a
  non-null field turns the parent into `null` and adds an entry to `errors`. The client may not
  handle a `null` parent.
- Enums are serialized by name; compare the names the client switches on.
- Errors usually come back with HTTP `200` and an `errors` array; see `error-contract.md`.
- Persisted queries or query allow-lists: a client operation that is not registered is rejected.

## 4. gRPC and protobuf specifics

- The wire identity of a field is its **number** and wire type. Compare the `.proto` each side
  compiles against, not only the field names. Reusing a deleted field number or changing a
  field's type is a breaking change; renaming a field is binary-compatible but changes JSON
  mapping (grpc-gateway, protojson).
- Check whether both sides compile the same `.proto` revision: a vendored copy, a separate
  module version, or a generated package committed at different times.
- Package and service names form the method path (`/pkg.Service/Method`). A package rename
  breaks every call.
- Enums: the zero value is the default when unset. A consumer that treats the zero value as a
  real state misreads unset fields. Unknown enum values are kept as numbers in proto3; check what
  the consumer's `switch` does with them.
- `optional`, wrapper types (`google.protobuf.StringValue`), and `oneof` change presence
  semantics. Compare them on both sides.
- Deadlines and metadata keys the server reads (`x-tenant-id`) are part of the contract when the
  handler depends on them.

## 5. Structural vs semantic mismatch

| Kind | Example |
|---|---|
| Structural | client calls `/api/orders`, server serves `/api/v1/orders`; client sends `accountId`, server decodes `account_id`; client sends a numeric string (`"42"`) where the server decodes an integer |
| Semantic | `GET /orders?status=open` where "open" means unpaid to the server and not shipped to the client; `duration` in minutes from the client and in seconds to the server; `page=1` meaning the first page to the client and the second page to a 0-based server |

## 6. Evidence to collect

```text
Server   route registration; middleware that reads headers or rewrites paths; handler; decode and encode types; defaults and validation
Client   URL construction and base-URL config; method; headers; request type and serializer; status checks; response decode type; fields read
Between  gateway, ingress, or proxy config in the repository; generated client and the spec it came from
Tests    handler tests with request fixtures; client tests with mocked responses. Do the mocked responses match what the handler writes?
```

## 7. Common false positives

- **Path prefixes added or stripped in between.** Check gateway, ingress, and router-group
  mounts before reporting a path mismatch.
- **Framework naming strategy.** A global snake_case setting makes a camelCase field name
  produce snake_case on the wire.
- **Fields the client does not use.** A response field the client never reads is not a mismatch.
- **Server defaults cover omitted fields.** A client that omits an optional field the server
  defaults is aligned, unless the default differs from what the client assumes.
- **Documentation disagreeing with both sides.** If the client and server code agree and only
  the OpenAPI file differs, the boundary is aligned. Record the documentation drift separately.
