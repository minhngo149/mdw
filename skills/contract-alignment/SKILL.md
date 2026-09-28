---
name: contract-alignment
description: Use when checking whether producers and consumers on both sides of a service boundary agree on a contract - HTTP/REST/GraphQL/gRPC APIs, RabbitMQ/Kafka/NATS/SQS events, webhooks, or shared payloads - including field names, types, money and timestamp units, enums, nullability, business states, and error codes; when an integration fails although each service compiles and passes its own tests; or when looking for contract drift, breaking changes, or backward-compatibility risk between services.
---

# Contract Alignment

Detect where producers and consumers across a service boundary disagree about a contract, by
comparing what the producer's code actually emits with what the consumer's code actually expects.
Record the result as reusable text: a Markdown artifact (source of truth) and an HTML rendering of
the same knowledge.

**Core principle:** code can be locally correct while a distributed contract is globally
inconsistent. Do not ask "is this service implemented correctly?" Ask "does this service agree
with the service on the other side of the boundary?"

```text
Producer code + consumer code  >  OpenAPI, README, comments
Wire contract                  >  Type and field names
Meaning                        >  Shape
Evidence                       >  Guess
```

## When to Use

- `/mdw contract-alignment billing` / `... InvoiceIssued` / `... checkout`
- "Do billing and ledger agree on the invoice event?" / "Who consumes this API, and what do they
  expect from it?"
- An integration misbehaves although both services compile and pass their own tests
- Before or after changing a payload, enum, route, topic, or error code: what breaks, and where?
- Scope can be a feature, endpoint, event, service, business flow, or the whole repository.
  Works for synchronous (HTTP, gRPC, GraphQL) and asynchronous (broker, webhook) boundaries.

Not for these questions, which belong to other MDW skills:

| Question | Skill |
|---|---|
| Where does the request or event travel? Retries, dead letters, transactions, failure paths? | `distributed-flow` |
| What executes in what order? | `sequence-flow` |
| How does the code use the database? | `database-performance` |
| Where can trust, identity, authorization, or data protection fail? | `security` |

Also not for generic code review, API design review, or redesigning the architecture.

## Evidence Labels

Every important claim carries exactly one label. These are the same three labels the other MDW
skills use.

| Label | Use when | Must include |
|---|---|---|
| `CONFIRMED` | You read the code, or in-repo config or schema, that shows it. A claim that the two sides agree or disagree is `CONFIRMED` only when you read **both** sides | file + symbol (one per side for an alignment claim); line range only if you read those lines |
| `INFERRED` | Strongly implied by code you read but not directly observed: a library's decoder default, runtime-selected config, a consequence of confirmed facts | the evidence + the reasoning step |
| `UNKNOWN` | The repository cannot decide: the other side is in another repository, a value comes from deploy-time config or a schema registry, the payload is built dynamically | what would resolve it |

- Documentation, comments, and names never make a claim `CONFIRMED`. When they disagree with
  code, code wins and the disagreement is recorded as **documentation drift**.
- Tests show what their author expected, not what runs. A test fixture that disagrees with the
  producer's real output is **test drift**.
- Never upgrade `INFERRED` to `CONFIRMED` without reading the code. Never fill an `UNKNOWN` with a
  typical value.

## Alignment Status

Status says whether the two sides agree. The label says how sure you are. A status of `UNKNOWN`
always carries the label `UNKNOWN`.

| Status | For one compared element | For a boundary (one Contract Matrix row) |
|---|---|---|
| `ALIGNED` | both sides agree, directly or through a mapping on the consumer's path | every compared element is `ALIGNED` |
| `PARTIALLY ALIGNED` | a mapping or branch covers only some of the producer's values | the primary case works (the consumer receives, decodes, and correctly acts on the producer's main output), but at least one element differs |
| `MISALIGNED` | the element differs and nothing reconciles it | a mismatch breaks the primary case: the message is not delivered, a key field is not read, the main state is not recognized, or a value is computed in the wrong unit |
| `UNKNOWN` | one side of the element is not in the repository, or the evidence cannot decide | the other side is outside the repository, or the pairing cannot be resolved |

## Severity

Severity describes the engineering impact of one mismatch. Never grade, score, or rank services,
and never use words like good, bad, poor, or excellent.

| Severity | Impact |
|---|---|
| `CRITICAL` | For the primary case, the consumer silently does nothing or the wrong thing: messages never delivered, identifier fields never read, the main state never recognized, money or quantities in the wrong unit |
| `HIGH` | Wrong behavior or a crash for a real subset of inputs, or an error read the wrong way so that a business operation ends in the wrong state |
| `MEDIUM` | A secondary value, state, or error is mishandled with limited, visible effect |
| `LOW` | A latent difference with no runtime effect today (a field the consumer reads but never acts on) |
| `OBSERVATION` | No mismatch today, but the alignment is fragile or undocumented: a mapping with no test, a lenient decoder hiding a difference, documentation drift |

`UNKNOWN` is a status and a label, never a severity. An `UNKNOWN` finding gets the severity of
its impact if the unverifiable side disagrees, written as `HIGH (potential)`.

## Workflow

Copy this checklist and complete it in order:

```text
[ ] 1. Scope
[ ] 2. Discover the repository and its components
[ ] 3. Discover boundaries
[ ] 4. Pair producers and consumers
[ ] 5. Extract each side's contract
[ ] 6. Compare: API, Event, Data, State, Error
[ ] 7. Compare meaning
[ ] 8. Detect drift
[ ] 9. Validate the evidence
[ ] 10. Write the findings
[ ] 11. Write the Markdown artifact, then the HTML artifact
[ ] 12. Self-check
```

### 1. Scope

A scope is a feature or business flow (`checkout`), a service (`billing-service`), an endpoint,
an event (`InvoiceIssued`), or the repository. The boundaries in scope are every boundary
where the scope is the producer or a consumer of a contract. For an event or endpoint, that is
the identifier's producers and **all** of its consumers. For a flow, it is every boundary on the
flow's path; if an MDW `distributed-flow` artifact exists for that flow
(`mdw/distributed-flow/<scope>-distributed-flow.md`), use its boundary list as a starting point and verify it.

Boundaries you notice but exclude (adjacent flows, contracts between two other services) go in
the Scope section with a one-line reason, so the reader can widen the scope. If the scope
matches several candidates (two services called `billing`), list them with evidence and ask. If
you cannot ask, pick the most likely one and state the choice.

### 2. Discover the repository and its components

Read what locates the deployables and their contracts: manifests (`go.mod`, `package.json`,
`pom.xml`, ...), entry points, `Dockerfile`, `docker-compose.yml`, Kubernetes or Helm manifests,
`Makefile`, `.proto`, OpenAPI, GraphQL SDL, event-definition packages, and tests. Classify each
component as service, worker, gateway, library, or external system. A service needs deployment
evidence (its own entry point and deployment unit). A shared DTO or event package is a
**library**, not a service. Do not scan the whole repository; follow what the scope touches.

### 3. Discover boundaries

A boundary is where data is serialized in one deployable and read in another (or in an external
system). Calls between modules of one process are not boundaries unless the payload crosses a
serialization step such as a queue.

| Producer signals | Consumer signals |
|---|---|
| HTTP route registration, gRPC server registration, GraphQL resolvers | HTTP clients (base URL + path), gRPC stubs, GraphQL operations, SDK calls |
| `Publish`, `Produce`, `Send`, `PublishEvent`, outbox relays | `Subscribe`, `Consume`, `QueueBind`, `HandleMessage`, listener annotations |
| Webhook senders | Webhook receiver routes |

### 4. Pair producers and consumers

Pair the two sides only through resolved routing identifiers: base URL + method + path,
exchange + routing key + binding, topic, subject, queue, webhook URL. Resolve constants and
config keys to values. Two services with similarly named types are **not** a boundary. Record
the pairing evidence. A side not in the repository is `UNKNOWN (not in this repository)`; never
invent it. Every consumer of a contract is its own pairing. Techniques:
`references/api-contract.md` section 1 and `references/event-contract.md` sections 1-2.

Assign roles by who **owns** the contract, not by which way the bytes move:

| Boundary | Producer (owns the contract; its code is the Actual Contract) | Consumer (its code is the Expected Contract) |
|---|---|---|
| HTTP, REST, GraphQL, gRPC | the server: its route, decode type, validation, responses, and errors | the client: the request it builds and the response and errors it reads |
| Broker message or event | the publisher: routing key or topic, envelope, payload | each subscriber: binding or subscription, decode type, handling |
| Webhook | the sender: URL, event type, payload, retry policy | the receiver: route, decode type, the responses it returns |

A request body flows from client to server, and the server still owns it: the client's request
is compared with what the server accepts.

### 5. Extract each side's contract

For each pairing, build a **contract sheet** (`references/contract-evidence.md` section 3):

- **Producer:** what goes on the wire: the serialized type, its tags or naming strategy, the
  values assigned, and the conditions under which fields are set, left empty, or omitted.
- **Consumer:** what it reads: the decode type and tags, and **every use site** that depends on a
  value (comparisons, arithmetic, lookups, dereferences), plus what it does with missing or
  unknown values.

The wire contract comes from the serializer, not the type name. A shared type does not prove a
shared wire contract (`contract-evidence.md` section 5).

### 6. Compare: API, Event, Data, State, Error

One boundary usually carries several categories. An event boundary has routing and envelope
(Event), fields (Data), status values (State), and sometimes failure signals (Error).

| Category | Compares | Reference |
|---|---|---|
| API | method, path, params, headers, bodies, status codes, pagination, auth context | `references/api-contract.md` |
| Event | routing, envelope, headers, event type, version, payload | `references/event-contract.md` |
| Data | names, types, units, IDs, money, time, enums, nullability, defaults | `references/data-contract.md` |
| State | state names, meanings, transitions, terminal states | `references/state-contract.md` |
| Error | status and error codes, error bodies, retryable vs permanent | `references/error-contract.md` |

Before reporting any mismatch, search the consumer's path for a mapping, adapter, custom
decoder, or serializer setting that reconciles it (`contract-evidence.md` section 4). Record
where you searched.

### 7. Compare meaning

Identical shape does not mean agreement. For every element that is structurally aligned, compare
what the value means when the producer emits it with what the consumer does because of it:
units, which entity an ID names, what a state implies has already happened. Cite the computation
on both sides (`Date.now()` against `datetime.fromtimestamp(x)`), not names or comments
(`references/data-contract.md` section 4).

### 8. Detect drift

Look for contracts that changed on one side only: a renamed field, key, route, or event; an added
enum value the consumer drops or rejects; a changed unit; a field made nullable or removed; a
changed meaning. Leads include "renamed from" comments, old constants still referenced,
versioned types, and `git log -S '<identifier>'`; the evidence is still the code on both sides.
Compare every test fixture and mock with the producer's real output (test drift), and every
documented contract with the code (documentation drift).

Discuss backward compatibility only for contracts you found, as current behavior, compatibility
implication, evidence, and possible resolution. Do not prescribe a universal versioning scheme.

### 9. Validate the evidence

A candidate is a contract-alignment finding when it passes this admission test:

```text
Actual Contract    = a value, name, route, state, or error the producer's code emits      (cited)
Expected Contract  = the consumer's code expects something different at the same element  (cited)
                     or: one side is outside the repository                               (UNKNOWN)
```

Then check each admitted finding: both sides cited with full repository-relative paths; mapping
searched; both sides reachable (the consumer is registered and deployed, the producer path really
emits the value); the consumer's decoder behavior considered (does the mismatch raise an error or
silently yield a zero value? `data-contract.md` section 2); label justified. Downgrade or drop
anything that fails.

A concern that has no producer value and consumer expectation to compare (a swallowed publish
error, a missing dead-letter queue, a race in one service, a missing auth check) is not a finding
here. Write it as one line under **Handed to other skills** in the Scope section, naming the MDW
skill that owns it, with no analysis.

### 10. Write the findings

Use the finding record in `references/contract-evidence.md` section 8 for every `MISALIGNED`,
`PARTIALLY ALIGNED`, material `UNKNOWN`, and `OBSERVATION`. Aligned elements appear in the matrix
and category sections, not as findings. Documentation drift and test drift are recorded in
Contract Drift; they become findings only as `OBSERVATION`, or as evidence inside the finding
they caused.

`Recommended Resolution` lists options tied to this mismatch (align the producer, align the
consumer, add an explicit mapping, accept both values during rollout, add a contract test), each
with its trade-offs. Say which option the evidence favors and why ("three consumers already read
`created_at`"), not which one is better in general. When a test would have caught the mismatch,
name the missing test; do not write it unless asked.

### 11. Write the artifacts

Read `references/contract-evidence.md` sections 9-11 first: diagram forms, section contents, and
the HTML skeleton.

**Output contract**

- Write exactly two files: `<scope-slug>-contract-alignment.md` and
  `<scope-slug>-contract-alignment.html`. The default location is `mdw/contract-alignment/` at
  the analyzed repository's root; a location the user gives takes precedence. If a file already
  exists, read it first and replace it only if it is an earlier MDW artifact for the same scope.
- Markdown is the source of truth. Write it first, then derive the HTML from it. The HTML adds no
  claims and drops none.
- Never produce Mermaid, `.mmd`, `.svg`, `.png`, `.drawio`, `.pdf`, `.docx`, images, inline
  `<svg>`, or `<canvas>`. Draw contracts as plain text in Markdown and with HTML/CSS in HTML.
- The HTML is one self-contained file with inline CSS: no external requests, no framework, no
  build step. It opens from disk and is fully readable with JavaScript disabled.

**Markdown structure.** Keep every heading, in this order. When a section does not apply, write
one line saying so and why (for example, "None: no gRPC boundary in scope"). Never pad a section.

```md
# Contract Alignment Analysis: <scope>

## 1. Scope
## 2. Services / Components
## 3. Boundaries
## 4. Producer / Consumer Map
## 5. Contract Matrix
## 6. API Contract Alignment
## 7. Event Contract Alignment
## 8. Data Contract Alignment
## 9. State Contract Alignment
## 10. Error Contract Alignment
## 11. Contract Drift
## 12. Alignment Findings
## 13. Evidence
## 14. Confirmed / Inferred / Unknown
## 15. Open Questions
```

Then reply to the user with the two paths, the matrix counts by status, the most severe findings
in one line each, and the number of `UNKNOWN` items. Do not paste the document into the reply.

### 12. Self-check

```text
[ ] Every boundary was paired through resolved routing identifiers, not similar names
[ ] Roles follow contract ownership: servers and publishers are producers, clients and subscribers are consumers
[ ] Every finding passes the admission test; other concerns are one line under Handed to other skills
[ ] Every finding cites both sides; CONFIRMED only where both sides were read
[ ] A mapping was searched for before every mismatch, and the search is recorded
[ ] Structurally aligned elements were also compared for meaning (units, IDs, states)
[ ] Test fixtures and documentation were compared with the producer's real output
[ ] Nothing outside the repository is described as fact; missing sides are UNKNOWN
[ ] Every evidence reference uses the full repository-relative path; no line number you did not read
[ ] No invented service, event, field, enum, error, version, or consumer
[ ] No findings that belong to distributed-flow, sequence-flow, database-performance, or security
[ ] No scores, rankings, or subjective labels; severity describes impact only
[ ] The HTML makes the same claims as the Markdown, in the same section order
[ ] Only .md and .html were written; no Mermaid, SVG, images, or diagram files
```

## Never Invent

Services, events, APIs, producers, consumers, schemas, fields, enum values, versions, errors,
line numbers, contracts, or relationships. When the repository cannot decide, write `UNKNOWN`.
When implied but not observed, write `INFERRED` and give the reasoning.

| Thought | Reality |
|---|---|
| "Both sides use `status: string`, so they agree" | Shape is not meaning. Compare the values each side uses and what the consumer does with them. |
| "The OpenAPI file describes the request, so that is the contract" | Documentation is a lead. Compare producer code with consumer code; record the drift. |
| "Both import the shared type, so the contract is aligned" | Check the decode site, serializer settings, module versions, routing, and units. |
| "The consumer's test passes" | Compare the test's fixture with what the producer really sends. |
| "`ACTIVE` vs `ENABLED` is a mismatch" | Search the consumer's path for a mapping first. Report only if none reconciles them. |
| "Both services define an `InvoiceStatus` type" | No boundary, no contract. Pair only through routing identifiers. |
| "The other side probably expects the same payload" | If the other side is not in the repository, its expectation is `UNKNOWN`. Say what would resolve it. |
| "A missing field would fail loudly" | Most decoders yield a zero value silently. Check the consumer's decoder before rating impact. |
| "The client sends the request body, so the client is the producer" | The server owns an API contract. The client is the consumer, even for request bodies. |
| "Also worth reporting: the publish error is ignored, there is no outbox or dead-letter queue, the duplicate check races" | No producer value vs consumer expectation, so not a finding here. One line under **Handed to other skills**. |
| "The consumer is outside the repo, so the severity is Unverified" | Severity is impact. Status and label are `UNKNOWN`; severity is the potential impact. |
| "The OpenAPI file is wrong in six places: a Medium finding" | Documentation drift goes in Contract Drift. It is an `OBSERVATION` at most, or evidence inside the finding it caused. |
| "Option A is the better one" / "The real fix is to share every DTO" | Give options with trade-offs, and say which one the evidence favors. No redesign. |
| "They should switch to schema versioning / gRPC" | Resolutions are options for this mismatch, not a redesign. |
| "Around line 40" | Cite lines only if you read them. Otherwise cite file + symbol. |

## References

Read each reference when its step needs it. They extend this workflow; they do not repeat it.

| File | Read when |
|---|---|
| `references/api-contract.md` | Steps 4-6 for HTTP, REST, GraphQL, or gRPC boundaries |
| `references/event-contract.md` | Steps 4-6 for broker, webhook, or message-bus boundaries |
| `references/data-contract.md` | Steps 5-7 for every boundary: serializers, decoder behavior, units, IDs, money, time, enums |
| `references/state-contract.md` | Step 6 when a boundary carries status or lifecycle values |
| `references/error-contract.md` | Step 6 when a boundary carries error responses, error codes, or retry hints |
| `references/contract-evidence.md` | Steps 5 and 8-11: evidence rules, contract sheet, mappings, shared types, tests, finding record, diagrams, both artifacts |
