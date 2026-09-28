---
name: distributed-flow
description: Use when asked how a specific feature, endpoint, API route, GraphQL resolver, gRPC method, CLI command, cron or scheduled job, webhook, message or event consumer, background worker, or business operation actually works in a codebase - tracing it end to end across handlers, services, databases, caches, message brokers (RabbitMQ, Kafka, NATS), and external APIs, or documenting its sync/async boundaries, persistence, transactions, retries, and failure paths.
---

# Distributed Flow

Reconstruct how one flow really executes, from its trigger to its last side effect, by reading
source code. Record the result as reusable text: a Markdown artifact (source of truth) and an
HTML rendering of the same knowledge.

**Core principle:** source code decides what the system does. Documentation, comments, names,
and conventions are leads to verify, never facts on their own.

```text
Source Code    >  Documentation  >  Assumption
Accuracy       >  Completeness
Evidence       >  Guess
Readable       >  Exhaustive
Reusable Text  >  Proprietary Diagram Format
```

## When to Use

- "How does `POST /orders` work?" / "What happens when a user clicks Pay?"
- "Trace the `order.created` event" / "What does the nightly settlement job do?"
- Onboarding, debugging, incident review, or design review of one flow
- Monoliths (most hops are `IN_PROCESS`) and distributed systems alike

Not for: whole-system surveys with no specific flow (list candidate entry points you found and
ask which one), reviewing a diff, or performance profiling.

## Evidence Labels

Every important claim carries exactly one label. These definitions hold for this skill, its
references, and both artifacts.

| Label | Use when | Must include |
|---|---|---|
| `CONFIRMED` | You read the code, or in-repo config/schema/manifest, that does it, on the path this flow executes | file + symbol; line range only if you read those lines |
| `INFERRED` | Strongly implied by code you read but not directly observed: library or framework defaults, runtime-selected wiring, consequences of confirmed facts ("commit, then publish, so the event can be lost") | the evidence + the reasoning step |
| `UNKNOWN` | The repository does not contain enough to decide: external systems, broker or infrastructure policy, deploy-time config, code in another repository | what would resolve it |

- Documentation, comments, and names alone never make a claim `CONFIRMED`. When they disagree
  with code, code wins and the disagreement is recorded as **documentation drift**.
- Never upgrade `INFERRED` to `CONFIRMED` without reading the code. Never fill an `UNKNOWN`
  with a "typical" value.
- An absence is `CONFIRMED` only for the places you inspected: "no retry in application code:
  client built in `NewPSPClient()` as a plain `http.Client`" is `CONFIRMED`; retries added
  by a proxy or service mesh remain `UNKNOWN`.
- In-repo config values are `CONFIRMED` as defaults; note when deployment can override them.

## Workflow

Copy this checklist and complete it in order:

```text
[ ] 1. Scope the flow
[ ] 2. Orient in the repository
[ ] 3. Locate the entry point
[ ] 4. Trace the call path
[ ] 5. Classify participants and boundaries
[ ] 6. Analyze persistence, cache, transactions, and state
[ ] 7. Analyze failure paths
[ ] 8. Build the flow model
[ ] 9. Write the Markdown artifact, then the HTML artifact
[ ] 10. Self-check
```

### 1. Scope the flow

Answer before reading code in depth:

```text
WHAT triggers the flow?        WHO triggers it?
WHERE does it enter?           WHAT business operation does it perform?
WHERE does it end?             (its last side effect, including async continuations)
```

If the request matches several flows (two `POST /orders` in different services, one event with
three consumers), list the candidates with evidence and ask. If you cannot ask, choose the most
likely one and state the choice in the Overview.

### 2. Orient in the repository

Identify only what this flow needs: language, framework, deployables (the processes that
run), entry points, and the infrastructure it touches (database, cache, broker, external APIs,
workers, relevant configuration). Start from manifests, entry points, and deployment files, then
follow dependencies outward. Do not scan the whole repository.
Techniques: `references/repository-analysis.md`.

### 3. Locate the entry point

Find where the trigger is **registered**: route table, resolver map, gRPC server registration,
CLI command tree, scheduler, or consumer subscription. A function with a matching name is not
enough. Record the middleware, interceptors, guards, and decorators that run before and after
the handler, in execution order.

### 4. Trace the call path

Follow the real execution path hop by hop. A common shape is:

```text
Entry point -> Handler/Controller -> Application/Service -> Domain -> Repository/Client -> DB / Cache / Broker / External
```

Treat that shape as a hint, not an expectation; follow what the repository actually does. At
each hop record the file, the symbol, and what it does. Resolve interfaces and dependency
injection to the implementation wired into this process. Confirm reachability: code that exists
but is not wired into this path (an unused outbox, a dead handler, a disabled feature flag) is
not part of the flow. Mention it only when documentation claims otherwise. Do not trace into
framework or library internals; record their behavior as `INFERRED` defaults.

### 5. Classify participants and boundaries

**Participants.** Each is one of: process, service, worker, module, package, library,
database, cache, broker, external system. Call something a *service* only when there is
deployment evidence (its own entry point, deployment manifest, or network address). Directory
names, class names, and README claims are not evidence. See `references/service-boundary.md`.

**Protocol.** Give each hop exactly one of:

```text
IN_PROCESS  HTTP  REST  GRAPHQL  GRPC  DATABASE  REDIS  RABBITMQ  KAFKA  NATS  WEBHOOK  OTHER_EXTERNAL_API
```

Name the concrete technology when the label is generic: `DATABASE (PostgreSQL)`,
`OTHER_EXTERNAL_API (AWS SQS)`. Use `REST` for resource-oriented HTTP APIs, `WEBHOOK` for HTTP
calls triggered by an event in another system (inbound or outbound), and `HTTP` for other HTTP
traffic.

**Mode.** Give each hop `SYNC` or `ASYNC`:

- `SYNC`: the caller waits for the receiver's result before continuing.
- `ASYNC`: the caller continues without the receiver's result. Examples: broker delivery,
  fire-and-forget goroutines, threads, or tasks, jobs picked up later.
- An "event" is not automatically `ASYNC`, because in-process event buses often dispatch
  synchronously. Publishing is usually a `SYNC` call to the broker followed by `ASYNC` delivery
  to the consumer. Record both halves.

For every hop that leaves the process, record **Caller, Receiver, Protocol, Mode, Purpose,
Evidence**. See `references/sync-flow.md` and `references/async-flow.md`.

### 6. Analyze persistence, cache, transactions, and state

- **Persistence:** for each meaningful operation, record Component, Store, Table/Collection,
  Operation (`INSERT`, `UPDATE`, `DELETE`, `SELECT`, `UPSERT`, `LOCK`), Transaction boundary,
  and Purpose.
- **Concurrency and consistency controls:** transactions, optimistic or pessimistic locking,
  unique constraints used for concurrency control, idempotency records, outbox, inbox. Report
  these only with evidence.
- **Cache:** operation, key pattern, TTL, and role (cache-aside, write-through, distributed
  lock, idempotency key, rate limit). Say where in the flow the cache participates.
- **Transactions:** exactly which operations are inside each database transaction and what runs
  outside it. Keep *database transaction* and *distributed business transaction* (saga,
  compensation, outbox, eventual consistency) separate.
- **State:** business state transitions, each with the function or event responsible.

See `references/transaction-flow.md`.

### 7. Analyze failure paths

The happy path is half of the analysis. For every hop, ask how it can fail, then follow the error
to where it is handled:

```text
Trigger -> Detection -> System behavior -> State change -> Response / recovery
```

Consider each of these that applies to the hops you found: validation, authentication,
authorization, database failure, external API failure, timeout, network failure, message
publish failure, message consume failure, retry, dead letter, rollback, compensation, fallback,
duplicate request, duplicate event, concurrency conflict. Label every step. Where the code does
not show the behavior, write `UNKNOWN`. See `references/failure-flow.md`.

### 8. Build the flow model

Before writing prose, write the **flow model**: a compact plain-text record of everything found
(entry, actor, participants, sync, async, persistence, state, failure, drift, unknowns). It
exposes gaps before you write, it becomes the appendix of the Markdown artifact, and other MDW
skills consume it. The grammar is in `references/text-flow-format.md`.

### 9. Write the artifacts

Read `references/text-flow-format.md` first. It defines the section contents, table columns,
text-diagram conventions, and the HTML skeleton.

**Output contract**

- Write exactly two files: `<flow-slug>.md` and `<flow-slug>.html`. The default location is
  `mdw/distributed-flow/` at the analyzed repository's root; a location the user gives takes
  precedence. If the file already exists, read it first and replace it only if it is an earlier
  MDW artifact for the same flow.
- Markdown is the source of truth. Write it first, then derive the HTML from it. The HTML adds
  no claims and drops none.
- Never produce Mermaid, `.mmd`, `.drawio`, `.png`, `.jpg`, `.svg`, inline `<svg>`, `<canvas>`,
  images, or any other diagram format. Draw flows as plain text in Markdown and with HTML/CSS in
  HTML.
- The HTML is one self-contained file with inline CSS. It makes no external requests (no CDN,
  fonts, or remote scripts), uses no framework, needs no build step, opens directly from disk,
  and is fully readable with JavaScript disabled.

**Markdown structure.** Keep every heading, in this order. When a section does not apply, write
one line saying so and why (for example, "None: no broker client is called on this path").
Never pad a section.

```md
# <Feature> Flow

## 1. Overview
## 2. Entry Point
## 3. Main Flow
## 4. Components
## 5. Synchronous Flow
## 6. Asynchronous Flow
## 7. Persistence
## 8. State Changes
## 9. Failure Flow
## 10. Transaction Boundary
## 11. Evidence
## 12. Confirmed / Inferred / Unknown
## 13. Open Questions
## Appendix: Flow Model
```

Then reply to the user with the two paths and a short summary: the entry point, the path in one
line, the most important risks or drift, and the number of `UNKNOWN` items. Do not paste the
whole document into the reply.

### 10. Self-check

```text
[ ] Every CONFIRMED claim cites a file and symbol you actually read; no line number you did not read
[ ] Every hop that leaves the process has Caller, Receiver, Protocol, Mode, Purpose, Evidence
[ ] Every async hop states ack, retry, and dead-letter behavior, or UNKNOWN
[ ] Every boundary has its failure behavior documented, or UNKNOWN
[ ] Nothing is called a service without deployment evidence
[ ] Every documentation claim contradicted by code is listed as drift
[ ] Code that exists but is not wired into this path is not described as part of the flow
[ ] Database transactions and distributed business transactions are kept separate
[ ] The HTML makes the same claims as the Markdown, in the same section order
[ ] Only .md and .html were written; no Mermaid, SVG, images, or diagram files
```

## Never Invent

Services, events, queues, topics, consumers, retry policies, timeouts, transactions, rollbacks,
database operations, or architecture patterns. When uncertain, write `UNKNOWN`. When implied
but not observed, write `INFERRED` and give the reasoning.

| Thought | Reality |
|---|---|
| "The README says it retries with backoff" | Docs are leads. Find the retry in code; otherwise record drift or `UNKNOWN`. |
| "There's an outbox table, so they use an outbox" | Existence is not use. Confirm it is called on this path and that a relay exists. |
| "It lives in `modules/billing`, so Billing is a service" | Directories are packages. A service needs its own deployable. |
| "The broker will retry failed messages" | Only if code or in-repo broker config says so. Otherwise `UNKNOWN`. |
| "Every production system has a DLQ and timeouts" | Report what this repository does. What is missing is `UNKNOWN` or a `CONFIRMED` absence. |
| "It's called an event, so it's async" | Check the dispatcher. In-process dispatch is often synchronous. |
| "It's inside the transaction because it comes after `BEGIN`" | Find the `COMMIT`. Check what runs after it and what cannot roll back. |
| "The request returned 201, so the flow is done" | Follow the async continuations, and check for errors that were logged and swallowed. |
| "Around line 40" | Cite lines only if you read them. Otherwise cite file + symbol. |
| "This would be clearer in Mermaid" | Output contract: plain text in `.md`, HTML/CSS in `.html`. |

## References

Read each reference when its step needs it. They extend this workflow; they do not repeat it.

| File | Read when |
|---|---|
| `references/repository-analysis.md` | Steps 2-4: orienting, finding entry points, tracing through DI, interfaces, and middleware |
| `references/service-boundary.md` | Step 5: deciding whether something is a service, process, worker, module, or external system |
| `references/sync-flow.md` | Steps 5 and 7: request/response chains, middleware order, timeouts, retries |
| `references/async-flow.md` | Whenever the flow publishes, consumes, enqueues, schedules, or spawns background work |
| `references/transaction-flow.md` | Step 6: any write, transaction, lock, idempotency record, outbox, or saga |
| `references/failure-flow.md` | Step 7 |
| `references/text-flow-format.md` | Steps 8-9: flow model grammar, text diagrams, section contents, HTML skeleton |
