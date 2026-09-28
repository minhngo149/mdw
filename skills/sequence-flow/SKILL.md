---
name: sequence-flow
description: Use when asked for the sequence flow, call chain, or step-by-step execution order of a specific endpoint, handler, function, method, use case, job, worker, or event consumer in a codebase - what calls what and in which order, through branches, loops, guard clauses, error returns, goroutines, threads or promises, transactions, and return values, at function level rather than service level.
---

# Sequence Flow

Reconstruct the actual execution sequence of one operation from source code: which functions
run, in what order, where control branches, loops, waits, fails, and returns. Record it as
reusable text: a Markdown artifact (source of truth) and an HTML rendering of the same
knowledge.

**Core principle:** the sequence is what the code executes, not what the architecture suggests.
Every step is a call, decision, or return that you read in source.

```text
Source Code    >  Documentation  >  Assumption
Accuracy       >  Completeness
Evidence       >  Guess
Readable       >  Exhaustive
Reusable Text  >  Proprietary Diagram Format
```

## When to Use

- "Sequence flow for `POST /orders`" / "Trace `CreateOrder()`" / "How does `ProcessRefund`
  execute?"
- "What happens when payment succeeds?" / "Sequence flow of the `order.created` consumer"
- Debugging "why does this return 500 here", reviewing the control flow of a hot path,
  onboarding to one operation

| The question is | Use |
|---|---|
| **What executes, in what order, and where does control move?** Functions, branches, loops, returns, goroutines, transactions as seen from the code | `sequence-flow` (this skill) |
| **Where does the system communicate?** Services, brokers, stores, protocols, delivery semantics, consistency across processes | `distributed-flow` |

The two compose. For one refund, distributed-flow shows `Billing API -> PSP -> PostgreSQL ->
RabbitMQ -> Notifier`. sequence-flow shows `RefundHandler.Create() ->
RefundService.ProcessRefund() -> FindPayment() -> [CARD] PSPClient.Refund() -> Save() (BEGIN ...
COMMIT) -> Publish() -> 201`, then an async boundary. When a step needs cross-process depth
(broker delivery, service classification, consumers in other processes), leave it to a
`distributed-flow` analysis instead of re-deriving it here.

Not for: whole-system surveys with no specific operation (list the candidate entry points you
found and ask which one), reviewing a diff, or performance profiling.

## Evidence Labels

MDW labels, with the same meaning as in distributed-flow. Every important claim carries exactly
one.

| Label | Use when | Must include |
|---|---|---|
| `CONFIRMED` | You read the code, or in-repo config/schema/migration, that does it, on the path this operation executes | file + symbol; line range only if you read those lines |
| `INFERRED` | Strongly implied by code you read but not directly observed: language, library, or framework semantics and defaults, runtime-selected wiring, consequences of confirmed facts | the evidence + the reasoning step |
| `UNKNOWN` | The repository does not contain enough to decide: external systems, infrastructure policy, deploy-time config, code in another repository | what would resolve it |

What they mean for a sequence:

- A **call** is `CONFIRMED` when you read the call site **and** resolved the callee to the
  implementation wired into this process. A name match is not resolution.
- An **order** between two steps is `CONFIRMED` when both are in code you read and one follows
  the other in straight-line code, or after a return, `await`, or join that links them. Steps in
  concurrent branches have no order. Say so; do not pick one.
- An **arm** is `CONFIRMED` when you read it. Whether this operation can reach it is a separate
  claim.
- An **absence** ("no retry", "nobody waits for this goroutine") is `CONFIRMED` only for the code
  you inspected. What infrastructure might add stays `UNKNOWN`.
- Documentation, comments, and names never make a claim `CONFIRMED`. When they disagree with the
  code, the code wins and the disagreement is recorded as **documentation drift**.

## Workflow

Copy this checklist and complete it in order:

```text
[ ] 1. Understand the request
[ ] 2. Find the entry point
[ ] 3. Trace the call chain
[ ] 4. Trace control flow
[ ] 5. Trace data flow
[ ] 6. Trace external boundaries
[ ] 7. Trace the return path
[ ] 8. Build the sequence model
[ ] 9. Validate against source
[ ] 10. Write the .md artifact
[ ] 11. Write the .html artifact
```

### 1. Understand the request

Resolve the request to **one operation with one starting point**. The input can be an endpoint,
function, class or method, feature, use case, condition ("when payment succeeds"), event, job,
worker, or file. If the request is thin, search the repository for the likely entry point before
asking. Ask only if several candidates remain equally plausible. If you cannot ask, choose the
most likely candidate and state the choice in the Overview. Mapping each input kind to a starting
point: `references/execution-tracing.md`.

### 2. Find the entry point

Find where the trigger is **registered** (route table, resolver, gRPC registration, CLI command,
scheduler, consumer subscription), then the function it points to, and the process that
registers it. A function with a matching name is not enough. Record the `ENTRY` block (trigger,
registration, file, function or method) and the wrapper chain in execution order: middleware,
interceptors, guards, decorators, and per-request transactions, with what each can do before and
after the handler. See `references/execution-tracing.md`.

### 3. Trace the call chain

Walk depth-first in execution order. Read each function on the path in full, and keep a trace
ledger (one line per step), so the order comes from the code, not from memory. For every call:

- **Resolve the callee.** Interfaces, DI, decorators, wrappers, callbacks, and registries point
  somewhere else. Confirm which implementation this process wires in.
- **Decide its depth.** Level 2 (function sequence) by default. Expand a function to Level 3 only
  when its internals decide the outcome. Never expand trivial helpers, standard-library calls,
  logging, or metrics.
- **Record meaningful functions**: Input, Output, Side effects, Calls (in order), Errors.

See `references/call-chain.md`.

### 4. Trace control flow

Capture every decision that changes what runs next: guard clauses, early returns, `if`/`else`,
`switch`/`case`, loops with their exits, error returns, exceptions, `defer`/`finally`, and
`panic`/`recover`. Do not flatten a branch into a linear story. Record every arm with a distinct
outcome and whether this operation can reach it. Trace each meaningful error from where it starts,
through every layer (returned, wrapped, mapped, swallowed, compensated), to the response the caller
receives. See `references/control-flow.md`, `references/branching.md`, and
`references/loops.md`.

### 5. Trace data flow

Track only the data that steers execution or forms the result, through each shape it takes
(request, command, entity, SQL parameters, rows, response) and the function that transforms it.
For each business state change, record from, to, the function responsible, and the later step at
which it becomes durable. See `references/execution-tracing.md`.

### 6. Trace external boundaries

Where execution leaves the process or stops waiting:

- **Database**: operation, table, which transaction it belongs to (or autocommit), and what the
  code checks in the result. Draw `BEGIN`, `COMMIT`, and every `ROLLBACK` exit inside the
  sequence. Never invent SQL.
- **External calls**: client function, protocol, target, request, response use, error mapping.
  Record timeouts and retries only as the code shows them.
- **Async boundaries**: publish, enqueue, spawned goroutines, threads, tasks, and un-awaited
  promises. The synchronous sequence continues with the caller's next statement. The other side is
  a **separate sequence** with its own entry point.
- **Concurrency**: parallel branches only with a spawn and a join you read, plus the join's
  semantics (waits for all, first error wins, the others keep running). Locks and critical
  sections where they change the order.

See `references/execution-tracing.md` and `references/async-boundary.md`. Cross-process
depth (which consumer receives a message, what the broker does on failure) belongs to
`distributed-flow`.

### 7. Trace the return path

Follow control and values back up to the original caller: each return value, each transformation
(row to entity to DTO to JSON), where the status or result is decided, and the exact call that
writes the response. Then record everything that still runs **after** the response is written:
deferred code, middleware on the way out, spawned work. See `references/control-flow.md`.

### 8. Build the sequence model

Before writing prose, write the **sequence model**: a compact, line-oriented record of the
entry, wrappers, participants, steps (with IDs, nesting, and labels), data, errors, drift,
code not on the path, and unknowns. It exposes gaps, becomes the Markdown appendix, and is what
other MDW skills consume. Grammar: `references/sequence-format.md`.

### 9. Validate against source

Walk the model again with the code open. This is a separate pass, not a memory check. Confirm
the entry point, every call and its resolved callee, the call order, branches, error paths,
return path, async boundaries, concurrency, database operations, transaction boundaries,
external calls, and evidence. Anything you cannot point at in code becomes `INFERRED` (with
reasoning) or `UNKNOWN` (with how to resolve it), or is removed. Full checklist:
`references/execution-tracing.md`.

### 10-11. Write the artifacts

Read `references/sequence-format.md` first. It defines step IDs, the text notation, section
contents, and the HTML skeleton, including the CSS lifeline diagram.

**Output contract**

- Write exactly two files: `<operation-slug>-sequence.md` and `<operation-slug>-sequence.html`.
  The default location is `mdw/sequence-flow/` at the analyzed repository's root; a location the
  user gives takes precedence. If a file already exists, read it first, and replace it only if it
  is an earlier MDW artifact for the same operation.
- Markdown is the source of truth. Write it first, then derive the HTML from it. The HTML adds no
  claims and drops none.
- Never produce Mermaid, `.mmd`, `.drawio`, `.png`, `.jpg`, `.svg`, inline `<svg>`, `<canvas>`,
  images, or any other diagram format. Draw sequences as plain text in Markdown and with HTML/CSS
  in HTML.
- The HTML is one self-contained file with inline CSS. It makes no external requests, uses no
  framework (no React, Vue, or Svelte), needs no build step, opens directly from disk, and is
  fully readable with JavaScript disabled.

**Markdown structure.** Use these titles in this order. Always include Overview, Entry Point, Main
Sequence, Function Call Chain, Error Paths, Return Path, Evidence, Confirmed / Inferred / Unknown,
and the appendix. Include the others when they have content. When you leave one out, give the
reason in one line in the Overview scope. Number the included sections consecutively.

```md
# <Operation> Sequence Flow

## 1. Overview
## 2. Entry Point
## 3. Main Sequence
## 4. Function Call Chain
## 5. Data Flow
## 6. Branches
## 7. Error Paths
## 8. Async / Concurrent Execution
## 9. Database / Transaction Sequence
## 10. Return Path
## 11. Evidence
## 12. Confirmed / Inferred / Unknown
## 13. Open Questions
## Appendix: Sequence Model
```

Then reply to the user with the two paths and a short summary: the entry point, the main path in
one line, the most important findings (order surprises, swallowed or unmapped errors, work after
the response, partial effects, drift), and the number of `UNKNOWN` items. Do not paste the whole
document into the reply.

## Never Invent

Function calls, callees, execution order, branches, retries, timeouts, concurrency, async
behavior, joins, database operations, SQL, transactions, rollbacks, or service boundaries. Never
assume `controller -> service -> repository` unless the repository implements it. When uncertain,
write `UNKNOWN`. When something is implied but not observed, write `INFERRED` and give the
reasoning.

| Thought | Reality |
|---|---|
| "A Mermaid `sequenceDiagram` is the standard way to draw this" | Output contract: plain text in `.md`, HTML/CSS in `.html`. No Mermaid, not even in a code block. |
| "Inline SVG keeps the HTML self-contained" | SVG is a diagram format. Use the CSS lifeline diagram from `sequence-format.md`. |
| "Go / Node / JVM semantics guarantee it, so it is `CONFIRMED`" | The code you read is `CONFIRMED`. A consequence you derive from language or library semantics ("the canceled context aborts the sibling call", "the channel closes and later publishes fail") is `INFERRED`, with the "because". |
| "This is clearly a race" / "this fails every time" | Describe the interleaving, label it `INFERRED`, and say what would confirm it. |
| "The consumer is part of the story, so its steps follow the response as steps 15-18" | After an async boundary, the steps belong to a separate sequence (`A1`, `A2`, ...). Continuing the numbering claims they run in order after the request. |
| "Both calls are in this function, so the first runs, then the second" | Inside `go`, `g.Go`, `Promise.all`, or `gather`, they run concurrently and have no order. Draw `PAR` and find the `JOIN`. |
| "The handler spawns the goroutine, and it finishes before the handler returns" | `go` does not wait. Draw it below the response write, marked not awaited, with what happens to its errors. |
| "It is an `async` function, so it is an async boundary" | An awaited call is sequential for the caller. Only an un-awaited call crosses a boundary. |
| "The error is returned, so the client gets the right status" | Check the error-to-response table, including its default. An unmapped error gets the default. |
| "It is called `validate`, so it runs" / "the stub is right there" | Only code reached from this path is in the sequence. List the rest under Not on this path. |
| "Controller calls service calls repository" | Follow the calls the code makes. Layer names are not evidence. |
| "The comment says the gateway retries" | Comments are leads. No retry in the code is a `CONFIRMED` absence for this repository; the gateway's behavior is `UNKNOWN`. |
| "The step table covers it, so I can skip the call tree and the function records" | They are required. They are what makes this a sequence rather than a flow narrative. |
| "Around line 40" | Cite lines only if you read them. Otherwise cite file + symbol. |

## References

Read each reference when its step needs it. They extend this workflow; they do not repeat it.

| File | Read when |
|---|---|
| `references/execution-tracing.md` | Steps 1-2 and 5-9: resolving the request, entry points, wrapper chains, the trace ledger, reachability, data and state, database and external calls, validation |
| `references/call-chain.md` | Step 3: resolving callees through interfaces, DI, wrappers, callbacks, registries, hidden calls; function records; Level 1/2/3 |
| `references/control-flow.md` | Steps 4 and 7: execution order, guards, error propagation, exceptions, `defer`/`finally`, panics, the return path, work after the response |
| `references/branching.md` | Step 4: any `if`, `switch`, type or table dispatch, feature flag; arm reachability; choosing paths |
| `references/loops.md` | Step 4: any loop, retry, pagination, polling, or per-item fan-out on the path |
| `references/async-boundary.md` | Step 6: any publish, enqueue, goroutine, thread, task, promise, `errgroup`/`WaitGroup`/`Promise.all`, lock, or event dispatch |
| `references/sequence-format.md` | Steps 8-11: step IDs, text notation, sequence model grammar, section contents, HTML skeleton |
