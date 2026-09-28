# Service Boundaries

Name every participant in the flow so that the boundaries in the document match the system's
**runtime** boundaries rather than its folder structure. A wrong boundary misleads readers about
latency, failure isolation, deployment, and consistency.

## Participant kinds

| Kind | What it is | Evidence that establishes it |
|---|---|---|
| process | A running OS process, one instance of a deployable | An entry point that starts a server, loop, or command |
| service | A deployable that owns a responsibility and is reached over the network | Its own entry point **and** its own deployment unit (Dockerfile, compose service, Kubernetes Deployment, serverless function), or a network address configured for it that callers use |
| worker | Something that consumes jobs, messages, or schedules instead of serving requests | A consumer loop or job framework. A separate deployable makes it a separate process; a goroutine or thread started by the API makes it part of the API process. |
| module | A cohesive unit of code inside one process (a group of packages, a Nest module, a Rails engine, a Java module) | Directory or build structure, called `IN_PROCESS` |
| package | A language namespace | Import path |
| library | Shared code linked into one or more deployables; never a runtime participant | Imported by entry points; has no entry point of its own |
| database | A datastore: SQL, document, key-value, search index | Client connection plus schema, migrations, or queries |
| cache | A store used to avoid recomputation, for locks, or for short-lived keys | Client plus get/set usage (often Redis or Memcached) |
| broker | RabbitMQ, Kafka, NATS, SQS, Pub/Sub, and so on | Client plus publish, consume, or declare calls |
| external system | Anything this repository does not own: third-party APIs, SaaS, other teams' services | A URL, SDK, or credentials in config, with no implementation in the repository |

Only processes, services, workers, databases, caches, brokers, and external systems are
**runtime participants**. Modules, packages, and libraries are code structure inside a process,
so calls between them are `IN_PROCESS`.

## Decision procedure

```text
Is the implementation outside this repository?
  yes -> external system (or database / cache / broker when it is infrastructure)
  no  -> Does it have its own entry point?
           no  -> module / package / library, reached IN_PROCESS
           yes -> Is there deployment evidence that it runs separately?
                    yes -> service (serves requests) or worker (consumes work)
                    no  -> process of the same codebase; deployment topology INFERRED or UNKNOWN
```

## Evidence strength

| Strength | Examples | Supports |
|---|---|---|
| Strong | A separate `main` plus a separate compose service or Kubernetes Deployment; callers use a client whose base URL is configured to point at it | `CONFIRMED` service or worker |
| Medium | A separate `main` but no deployment manifests; one binary started with `--mode=worker` or `ROLE=consumer` | Separate process, `INFERRED` |
| Weak, never enough alone | A directory named `services/`, a class named `PaymentService`, a README that says "microservice", a Go module or package name | Nothing; leads only |

## Traps

- **`services/` directories.** Usually an application-service layer inside one process.
- **"Client" naming.** An `InventoryClient` that calls another in-process module is
  `IN_PROCESS`. An `InventoryClient` that sends HTTP to a URL crosses a process boundary. Find
  the receiving route; it may be in this repository.
- **One binary, several roles.** Subcommands, mode flags, or role environment variables produce
  separate processes when deployed with different arguments. Manifests show this; without them
  the topology is `INFERRED`.
- **Worker inside the API process.** A consumer goroutine or thread started in the API's `main`
  shares the API's lifecycle, scaling, and crash domain. Label it "worker (in API process)".
- **External system mistaken for an in-repo service.** A README may list "Fraud Service" when
  the repository only contains a client for a third-party scoring API. Name the participant
  after what the code calls: the config key or host (for example `RiskScore (RISK_API_URL)`).
  Do not guess a vendor.
- **Service mesh and sidecars.** They can add retries, timeouts, and mTLS outside the code.
  Report them only from manifests in the repository; otherwise `UNKNOWN`.
- **Shared database.** Two deployables that write the same tables are coupled through the
  database even if they never call each other. Record it.

## Naming participants

- Name processes after their deployable, with the evidence: `Order API (cmd/api)`,
  `Notifier (compose service: worker)`.
- Name in-process components by code symbol: `OrderService.CreateOrder`.
- Name infrastructure by technology and the config that locates it: `PostgreSQL
  (DATABASE_URL)`, `RabbitMQ (AMQP_URL)`.
- Use the names the code uses. Do not rename participants to match documentation.

## Recording boundaries

Every hop that crosses a runtime participant boundary gets one row:

```text
| # | Caller | Receiver | Protocol | Mode | Purpose | Evidence | Label |
|---|--------|----------|----------|------|---------|----------|-------|
| 1 | Booking API (cmd/api) | PostgreSQL (DB_DSN) | DATABASE (PostgreSQL) | SYNC | lock seats, insert booking | internal/booking/repo.go Reserve() | CONFIRMED |
| 2 | Booking API (cmd/api) | PSP (PSP_URL) | REST | SYNC | charge card | internal/booking/psp.go Pay() | CONFIRMED |
| 3 | Booking API (cmd/api) | Kafka topic booking.confirmed | KAFKA | ASYNC | notify ticketing | internal/booking/events.go Emit() | CONFIRMED |
```

In a monolith this table is short: the database, cache, broker, and external APIs. That is a
correct result. Do not pad it with module-to-module calls; describe those in the main flow
steps.
