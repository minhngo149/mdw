---
name: distributed-flow
description: Use when asked what actually happens when a specific request, endpoint, job, cron, webhook, event, message consumer, or business operation runs across a system - which services, workers, databases, caches, message brokers (RabbitMQ, Kafka, NATS), gRPC or HTTP calls, and external APIs it touches, where it is synchronous or asynchronous, and where it can fail.
---

# Distributed Flow

## GOAL

Answer one question from the repository's code: **"What actually happens when this runs?"**

The main output is the **FLOW**: nodes and connections, from the entry point to the last side
effect. Everything else explains that flow.

The input is a scope: `/mdw:distributed-flow POST /orders`, a job, an event, a consumer, or a
business operation.

## WHAT

Trace only the requested flow:

- where it starts (the registered entry point) and where it ends (the last side effect)
- the services, workers, and infrastructure it touches: database, cache, broker, gRPC/HTTP, external API
- the protocol and mode (sync or async) of every connection
- the important failure points

A component that exists in the repository, its config, or its docs but is not on this path is
not in the flow.

## WHY

A flow that looks simple from one service hides dependencies across boundaries. The trace
exposes service boundaries, where sync ends and async begins, database and cache writes,
message hops, external calls, and what breaks when one of them fails.

## HOW

Evidence priority: **Source code > Tests > Configuration > Documentation > Assumption.**
Names and docs are leads, never proof. When docs contradict code, the code wins and the
contradiction is written down.

1. **Entry point.** Find where the trigger is *registered*: route table, resolver, gRPC server,
   CLI command, scheduler, webhook route, or consumer subscription. A similar function name is
   not an entry point. If several match, list them and ask, or pick one and say so.
2. **Follow the calls.** Read each hop: handler → service → repository / client. Resolve
   interfaces and dependency injection to the implementation actually wired in. Note early
   returns that end the flow.
3. **Record every boundary.** Each time code leaves the process, record the target, protocol,
   mode, and the file + symbol that does it: query or transaction, cache get/set/delete,
   publish (exchange, topic, routing key, subject), HTTP or gRPC call (target from its config key).
4. **Cross async boundaries by identifier.** A publish connects to a consumer only if the same
   topic, routing key, queue, subject, or job name appears in the publisher and in the
   consumer's registration. Then continue inside the consumer. No match in the repository →
   the consumer is `UNKNOWN`. An "event" dispatched in-process may be synchronous: check the
   dispatcher.
5. **Stop** at the last side effect, or where the flow leaves the repository (external API,
   another repo's service). Their internals are `UNKNOWN`.
6. **Failure.** At each boundary, read what the code does when the call fails (see FAILURE).

A node is a **service** or **worker** only if it has its own deployable: its own `main`,
manifest, Dockerfile, or compose/k8s service. Layers inside one process are modules; when you
show them, their connection is `In-process`.

### Evidence labels

| Label | Meaning | Must include |
|---|---|---|
| `CONFIRMED` | Read in code or in-repo config on this path | file + symbol |
| `INFERRED` | Implied by code you read: a library default, a consequence of confirmed facts | the reasoning |
| `UNKNOWN` | The repository cannot tell: broker policy, external system, other repo, deploy config | what would resolve it |

In the output, a statement without a label is `CONFIRMED` and its file + symbol is in HOW.
Label everything else inline.

## FLOW

Draw nodes and connections as plain text, top to bottom, in execution order. Label every arrow
with protocol and mode. Use `├──→` / `└──→` for fan-out.

```text
[Client]
    ↓ HTTP / Sync
[Order API]
    ↓ In-process / Sync
[Order Service]
   ├──→ [PostgreSQL]         SQL / Sync
   ├──→ [Redis]              Redis / Sync
   └──→ [RabbitMQ]           AMQP / Async
              ↓ AMQP / Async
       [Payment Worker]
              ↓ HTTP / Sync
       [Payment Provider]
```

- Node types: `Client`, `API`, `Service`, `Worker`, `Cron`, `Webhook`, `Database`, `Cache`,
  `Queue`, `External`. The node name carries the technology (`PostgreSQL`, `Redis`, `RabbitMQ`).
- Protocols: `HTTP`, `gRPC`, `SQL`, `Redis`, `AMQP`, `Kafka`, `NATS`, `In-process`, or the SDK
  the code uses.
- Mode: `Sync` (the caller waits for the result) or `Async` (it does not).

## TRADE-OFF

At most four lines, each a trade-off visible in this flow: `choice → benefit → cost`.

```text
Publish after commit → API stays decoupled from payment → event lost if publish fails
Sync PSP call in worker → simple → one slow PSP call blocks the queue (prefetch 1)
```

## FAILURE

The failure points that matter, in flow order, three to eight rows. For each, what the code does
(rethrow, status code, log and swallow, rollback, ack / nack / requeue, retry, timeout,
fallback) and where the flow ends up.

Retry, timeout, fallback, rollback, dead-letter, and compensation appear only when the code has
them. Otherwise write `none found in code`; what infrastructure might add (broker policy, proxy,
mesh) is `UNKNOWN`.

```text
[RabbitMQ] → [Payment Worker] → processing failure → nack, no requeue → UNKNOWN (no DLX in code)
```

## OUTPUT

Two files, by default in `mdw/distributed-flow/` at the analyzed repository's root (a location
the user names wins): `<scope>-distributed-flow.md` and `<scope>-distributed-flow.html`, where
`<scope>` is a kebab-case slug (`post-orders`).

### Markdown

The source of truth, under 120 lines. Every line describes what the flow does or what the
repository cannot tell. It has exactly these sections, in this order:

````md
# Distributed Flow: <scope>

## GOAL
One line: the question this flow answers.

## WHAT
Entry point, where the flow ends, components involved.
Not in this flow: components the repo, config, or docs suggest but this path does not reach.

## WHY
2-4 bullets: what the trace exposes (hidden dependency, sync/async boundary, docs vs code).

## HOW
Numbered steps in execution order, each citing `file` + symbol.

## FLOW
```text
<diagram>
```

### Nodes
| Node | Type | Purpose |
|---|---|---|

### Connections
| From | To | Protocol | Mode |
|---|---|---|---|

## TRADE-OFF
<= 4 lines.

## FAILURE
| Where | Failure | What the code does | Result |
|---|---|---|---|
````

The FLOW diagram is a ```` ```text ```` block, never Mermaid.

### HTML

The same content as the Markdown, adding no claims and dropping none. One self-contained file:
inline CSS, little or no JavaScript, no external requests, opens from disk, readable in light
and dark mode. The flow is drawn with HTML elements and CSS only:

```html
<div class="flow">                                   <!-- column, top to bottom -->
  <div class="node api">Order API<small>API</small></div>
  <div class="edge sync">↓ In-process · Sync</div>  <!-- solid line -->
  <div class="branch">                               <!-- fan-out: lanes side by side -->
    <div class="lane"><div class="edge sync">↓ SQL · Sync</div><div class="node database">PostgreSQL<small>Database</small></div></div>
    <div class="lane"><div class="edge async">↓ AMQP · Async</div><div class="node queue">RabbitMQ<small>Queue</small></div></div>
  </div>
</div>
```

One CSS class per node type, sync edges solid, async edges dashed. After the flow, the same
GOAL / WHAT / WHY / HOW text and the Nodes, Connections, and Failure tables.

Write only these two files: no Mermaid, `.mmd`, `.svg` or inline `<svg>`, `<canvas>`, images,
`.drawio`, `.pdf`, or `.docx`.

### Reply

The two paths, the flow in one line, the top failure points, and the number of `UNKNOWN` items.

## RULES

| Thought | Reality |
|---|---|
| "There's a `PaymentService`, so the flow calls it" | Find the call. Existence is not use. |
| "The README says it retries 3 times" | Find the retry in code. Otherwise `none found in code`, and note the docs contradict it. |
| "docker-compose has Kafka, so it's in the flow" | Only if code on this path produces or consumes it. |
| "That worker consumes from this exchange" | Only if its binding matches what the publisher sends. |
| "The broker will redeliver or dead-letter it" | Only if code or in-repo config says so. Otherwise `UNKNOWN`. |
| "It returned 201, the flow is done" | Follow the async continuation to its last side effect. |
| "While I'm here: fixes, security, validation" | Not this document. The flow is the deliverable. |
| "A sequence diagram would be clearer" | Plain text in `.md`, HTML/CSS in `.html`. |
