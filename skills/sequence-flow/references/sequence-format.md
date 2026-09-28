# Sequence Format

MDW's conventions for sequence artifacts: evidence references, step IDs, plain-text sequence
notation, the sequence model, the Markdown sections, and the HTML rendering. People and other
MDW skills can read any sequence artifact without a translation step.

Diagrams are plain ASCII text in Markdown and HTML/CSS in HTML. There is no Mermaid, no SVG, no
images, and no diagram files. ASCII renders the same way in every terminal, font, and diff.

## Contents

1. Evidence references
2. Step IDs
3. Text notation
4. Diagram patterns
5. Sequence model
6. Markdown artifact: section contents
7. HTML artifact: rules and skeleton

## 1. Evidence references

```text
path/to/file.ext:START-END Symbol     when you read those exact lines
path/to/file.ext Symbol               otherwise
```

- Paths are relative to the repository root and use forward slashes.
- Symbol forms: Go `Type.Method` or `Func`; Java, Kotlin, and C# `Class#method`; JS/TS
  `Class.method`, `functionName`, or `export POST`; Python `module.function` or
  `Class.method`; Ruby `Class#method`.
- Take line numbers only from what you read (the Read tool's line prefixes, or `grep -n`).
  Never estimate them.
- Labels are spelled out in upper case: `CONFIRMED`, `INFERRED`, `UNKNOWN`. An `INFERRED`
  claim carries its reasoning ("because ..."). An `UNKNOWN` carries how to resolve it
  ("resolve by ...").

## 2. Step IDs

Every step gets one ID. The same ID is used in the diagrams, the tables, the HTML, and the
sequence model, so a reader can follow one step across all of them.

| ID | Meaning |
|---|---|
| `1`, `2`, `3` ... | Main-path steps in execution order. Nesting is shown by indentation in the call tree, not by the ID. |
| `6a`, `6b`, `6c` | Arms of the decision at step 6, or the concurrent branches of a `PAR` at step 6 |
| `6a.1`, `6a.2` | Steps inside arm `6a` |
| `8.1`, `8.2` | Steps inside the body of the loop at step 8 |
| `4x`, `4x2` | Error exits from step 4 |
| `A1`, `A2` ... / `B1` ... | Steps of a separate sequence: the other side of an async boundary, or spawned background work. Each separate sequence has a title: `Sequence A: refund.completed consumer (cmd/notifier)`. |

Number the happy path first, then give arms and error exits the IDs of the step they leave from.

## 3. Text notation

| Notation | Meaning |
|---|---|
| `Symbol()` on its own line | code that holds control at that point |
| `[Name]` | a participant outside this process's code: client, database, cache, broker, external system |
| `\|` then `v` | control passes downward. The label on the `\|` line is the call, or a return. |
| `<- value` | a return label: control goes back to the caller with `value` |
| `x-->` | error exit: `x--> ErrNotFound`, `x--> 404 not_found` |
| `+-- [guard]` | a branch arm, with its guard written as in the code |
| `+-->` | a call made from a branch or fan-out point |
| `PAR` ... `JOIN` | concurrent branches, and the statement that waits for them |
| `+~~>` | spawn or handoff that the caller does not wait for |
| `:` then `v` | asynchronous continuation, reading downward |
| `==== ASYNC BOUNDARY ====` | everything below runs later, somewhere else, as a separate sequence |
| `LOOP` | a loop; its body is indented under it |
| `BEGIN`, `COMMIT`, `ROLLBACK` | transaction boundaries |
| `DEFER` | deferred, `finally`, or cleanup code; draw it where it runs, at scope exit |
| `(?)` | unknown hop, explained in the Unknown list |
| `(INFERRED)` | a hop that is not `CONFIRMED`, marked inline |

Rules:

- Time flows downward. The line that comes lower in a diagram executes later.
- Keep every text block to 120 columns. Split a diagram at a natural seam (entry, service,
  persistence, response) when it passes about 60 lines. Draw one diagram per question instead of
  one diagram for everything.
- Draw a return when it carries a value that the caller branches on or transforms. Fold
  pass-through returns into the next line.

## 4. Diagram patterns

The examples below use one hypothetical refund operation. Copy the notation, not the content.

**Main sequence**: who holds control, in order (Level 2).

```text
[Client]
   |  POST /refunds
   v
RequireAuth()                               x--> 401 unauthorized
   |  next.ServeHTTP(w, r)
   v
RefundHandler.Create()                      x--> 400 invalid_json
   |  ProcessRefund(ctx, cmd)
   v
RefundService.ProcessRefund()
   |  FindPayment(ctx, cmd.PaymentID)
   v
RefundRepository.FindPayment()
   |  SELECT payments WHERE id = $1
   v
[PostgreSQL]
   |  <- row | no rows
   v
RefundRepository.FindPayment()              x--> ErrNotFound (no rows)
   |  <- *Payment
   v
RefundService.ProcessRefund()               x--> ErrNotRefundable (status != SETTLED)
   +-- [CARD]          +--> PSPClient.Refund()        (Branches B1)
   +-- [STORE_CREDIT]  +--> Ledger.Credit()
   +-- [default]       x--> ErrUnsupportedMethod
   |  Save(ctx, refund)
   v
RefundRepository.Save()                     T1: BEGIN ... COMMIT (Transactions)
   |  <- nil
   v
RefundService.ProcessRefund()
   |  Publish(ctx, "refund.completed", evt)
   v
[RabbitMQ]
   |  <- err: logged, not returned
   v
RefundService.ProcessRefund()
   |  <- *Refund
   v
RefundHandler.Create()
   |  <- 201 Created {id, status}
   v
[Client]
   :
==== ASYNC BOUNDARY: refund.completed is delivered later, to cmd/notifier (Sequence A) ====
```

**Call tree**: nesting means "called by the line above it", and top to bottom means execution order.

```text
1    RequireAuth()                                  internal/http/auth.go         x-> 401
2    `-- RefundHandler.Create()                     internal/http/refund.go       x-> 400
3        |-- RefundService.ProcessRefund(ctx, cmd)  internal/refund/service.go
4        |   |-- RefundRepository.FindPayment()     internal/refund/repo.go       DB SELECT payments  x-> ErrNotFound
5        |   |-- IF p.Status != SETTLED                                           x-> ErrNotRefundable
6        |   |-- SWITCH p.Method                                                  (Branches B1)
6a       |   |   |-- [CARD] PSPClient.Refund()      internal/psp/client.go        EXT HTTP POST /v1/refunds
6b       |   |   |-- [STORE_CREDIT] Ledger.Credit() internal/ledger/ledger.go     DB INSERT ledger_entries
6x       |   |   `-- [default]                                                    x-> ErrUnsupportedMethod
7        |   |-- Refund.Complete()                  internal/refund/model.go      STATE REQUESTED -> COMPLETED
8        |   |-- RefundRepository.Save()            internal/refund/repo.go       TX T1
9        |   |-- Publisher.Publish()                internal/events/publisher.go  ASYNC ~> RabbitMQ
10       |   `-- return r
11       `-- writeJSON(w, 201, toRefundResponse(r)) internal/http/respond.go
```

**Branch**: one decision, every arm with a distinct outcome.

```text
B1  RefundService.ProcessRefund(): switch p.Method                  internal/refund/service.go
    |
    +-- [CARD]
    |     |  PSPClient.Refund(ctx, p.ChargeID, amount)
    |     v
    |   [PSP]  HTTP POST {PSP_URL}/v1/refunds
    |     x--> 402 -> ErrPSPDeclined
    |     x--> 5xx, network error, timeout -> ErrPSPUnavailable
    |
    +-- [STORE_CREDIT]
    |     +--> Ledger.Credit(): INSERT ledger_entries (autocommit, outside T1)
    |
    +-- [default]
          x--> ErrUnsupportedMethod   not reachable: validate() accepts only CARD, STORE_CREDIT
```

A binary decision can use the question form:

```text
validate(cmd) failed?
   +-- YES  x--> return ErrValidation
   +-- NO   continue to step 4
```

**Loop**: what is iterated, the bound, the body, and every exit.

```text
8   RefundRepository.Save()   (inside T1)
    |
    +-- LOOP for each l in r.Lines          bound: validate() allows 1..20 lines
    |     8.1  +--> INSERT refund_lines
    |          x--> error: return err -> DEFER tx.Rollback() undoes lines 1..k-1
    |
    +--> UPDATE payments SET refunded_cents = refunded_cents + $1
```

**Concurrent branches and join**: only with spawn and join evidence.

```text
RefundService.ProcessRefund()
   |
   PAR  errgroup.WithContext(ctx)
   +-- 4a  FraudClient.Score()       HTTP POST /score
   +-- 4b  LimitRepo.ForCustomer()   SELECT refund_limits
   JOIN g.Wait(): waits for both; returns the first error; the shared ctx is canceled on first error
   |
   v
RefundService.ProcessRefund()   continues with both results
```

**Spawned work**: the caller does not wait.

```text
RefundHandler.Create()
   |  <- 201 Created (response already written)
   |
   +~~> go h.notify(context.Background(), r)       Sequence B, not awaited
   |      error: discarded     panic: not recovered in this goroutine
   v
return
```

**Async boundary**: the sequence stops at the handoff. The other side is its own sequence.

```text
RefundService.ProcessRefund()
   |  Publish(ctx, "refund.completed", evt)       sync call to the broker client
   v
[RabbitMQ]  exchange billing, routing key refund.completed
   :
==== ASYNC BOUNDARY: delivered later, to cmd/notifier ====
   :
   v
A1  Notifier.Handle()     registered by cmd/notifier/main.go Consume("notify.refund-completed")
```

**Transaction**: inside the sequence, with every exit.

```text
T1  RefundRepository.Save()                                   internal/refund/repo.go
    |  BEGIN                        pool.Begin(ctx)
    |  DEFER tx.Rollback(ctx)       runs at every exit; no-op after a successful COMMIT
    |  INSERT refunds
    |  LOOP INSERT refund_lines
    |  UPDATE payments              rows affected not checked
    |  COMMIT                       tx.Commit(ctx)
    |
    x--> error before COMMIT: return err -> DEFER ROLLBACK -> nothing from T1 persists
    x--> COMMIT error: returned to caller; outcome ambiguous (INFERRED)
Outside T1: 6a PSP refund and 6b INSERT ledger_entries already happened; ROLLBACK does not undo them.
```

**Error path**: from origin to response, one line per layer.

```text
E2  PSP declines the refund
  origin    6a  HTTP 402 -> ErrPSPDeclined           CONFIRMED  internal/psp/client.go PSPClient.Refund()
  up        3   returned unchanged                   CONFIRMED  internal/refund/service.go RefundService.ProcessRefund()
  up        2   passed to writeError(w, err)         CONFIRMED  internal/http/refund.go RefundHandler.Create()
  mapped    -   ErrPSPDeclined -> 402                CONFIRMED  internal/http/respond.go statusFor()
  state     -   nothing written; T1 never starts     CONFIRMED  (T1 is step 8, after 6a)
  response  -   402 {"error":"payment_declined"}     CONFIRMED  internal/http/respond.go statusFor()
```

**Return path and data flow**: the values that come back, and each transformation.

```text
[PostgreSQL]  refunds row
   |  <- rows.Scan -> refund.Refund              RefundRepository.FindByID()
   v
RefundService.ProcessRefund()
   |  <- *refund.Refund
   v
RefundHandler.Create()
   |  toRefundResponse(r) -> RefundResponse{id, status, amount_cents}
   |  writeJSON(w, 201, resp)
   v
[Client]  201 {"id": ..., "status": "COMPLETED", "amount_cents": ...}
```

**State mutation**: tie each change to its step, and say when it becomes durable.

```text
Refund.Status
  REQUESTED --7 Refund.Complete()--> COMPLETED          memory only
                                        |
                                        +-- persisted at 8 (INSERT refunds, T1)
                                        x-- lost if T1 fails; the PSP refund from 6a remains
```

## 5. Sequence model

The sequence model is the compact, line-oriented record of the trace. It goes at the end of the
Markdown artifact under `## Appendix: Sequence Model`, in a `text` code block. Build it before
writing prose: gaps show up as lines you cannot fill in. Other MDW skills parse it, so keep to
the grammar.

```text
SEQUENCE: <operation name>
SOURCE: <repository> @ <short commit sha | "uncommitted"> (<YYYY-MM-DD>)
LEVEL: 2 [; level 3 in <Symbol>, <Symbol>]
ENTRY
  <TRIGGER> => <file> <Symbol>                                              <LABEL>
WRAPPERS
  <n>. <file> <Symbol>: <what it does>; exits: <early exits | none>         <LABEL>
PARTICIPANTS
  <name>  <kind>  <file | config key>                                       <LABEL>
STEPS
  <id>  <caller> -> <callee>: <call>                                        <LABEL>  <file> <Symbol>
  <id>  <callee> <- <value>                                                 <LABEL>
  <id>  x> <error | response>: <condition>                                  <LABEL>
  <id>  IF | SWITCH <condition as in code>                                  <LABEL>
  <id>    [<guard>] <caller> -> <callee>: <call>                            <LABEL>
  <id>  LOOP <over what> (bound: <bound | unbounded | UNKNOWN>)             <LABEL>
  <id>  PAR <mechanism>                                                     <LABEL>
  <id>  JOIN <mechanism>: <semantics>                                       <LABEL>
  <id>  <caller> ~> <target>: <spawn | publish | enqueue>, awaited: <no>    <LABEL>
  <id>  TX <tx id> BEGIN | COMMIT | ROLLBACK | DEFER ROLLBACK               <LABEL>
  <id>  STATE <Entity.field> <FROM> -> <TO> (<memory | persisted by <id>>)  <LABEL>
  ---   ASYNC BOUNDARY -> SEQUENCE <letter>
SEQUENCE <letter>: <name>
  ENTRY ... STEPS ...  (same grammar)
DATA
  <type> -> <type>  via <Symbol>                                            <LABEL>
ERRORS
  E<n>  <origin id> <error> -> <propagation> -> <response>; state: <what persists>   <LABEL>
DRIFT
  <documentation claim> (<doc file>)  vs  <what the code does>              <LABEL>
NOT ON PATH
  <file> <Symbol>: <why it does not execute on this path>                   <LABEL>
UNKNOWN
  <item>  (resolve by <how>)
```

Rules:

- One fact per line. Indent two spaces per nesting level under `IF`, `SWITCH`, `LOOP`, `PAR`, and
  `TX`.
- `->` is a synchronous call, `<-` a return, `~>` a handoff the caller does not wait for, `x>` an
  error exit, `=>` a trigger pointing at its code.
- Participant `kind` is one of: actor, code, database, cache, broker, external system, worker.
- Omit a block that has nothing in it, except `UNKNOWN`: always include it, with `none` if
  nothing is unknown.

## 6. Markdown artifact: section contents

Use this order and these titles. Always include Overview, Entry Point, Main Sequence, Function
Call Chain, Error Paths, Return Path, Evidence, Confirmed / Inferred / Unknown, and the appendix.
Include the other sections when they have content. When you leave one out, say why in one line in
the Overview scope (for example: "No async or concurrent execution: no spawn, publish, or
enqueue in the traced functions"). Number the included sections consecutively. Every table that
holds claims has a `Label` column.

| Section | Contents |
|---|---|
| Title | `# <Operation> Sequence Flow` |
| Overview | A metadata block (repository, commit, analysis date, operation, level, `Generated by MDW sequence-flow`). Scope: what is traced, what is excluded, and the omitted sections with the reason for each. A summary paragraph. **Key findings**: 3-7 bullets, most important first. Candidates: order surprises, errors that are swallowed or not mapped, work done after the response is written, partial effects, arms that cannot be reached, drift. Then the counts of `CONFIRMED`, `INFERRED`, and `UNKNOWN` items. |
| Entry Point | `ENTRY` block: trigger, registration (evidence), file, function or method. Then the wrapper chain (middleware, interceptors, decorators, per-request transactions) in execution order, with what each wrapper can do before and after the handler. |
| Main Sequence | The main sequence diagram (section 4), then the step table: `# \| Caller \| Callee \| Call / action \| Kind \| Evidence \| Label`. Kind is one of `CALL`, `RETURN`, `DECISION`, `LOOP`, `PAR`, `JOIN`, `DB`, `EXT`, `SPAWN`, `PUBLISH`, `STATE`, `ERROR`. |
| Function Call Chain | The call tree. Then one record for each meaningful function: `Function / Input / Output / Side effects / Calls (in order) / Errors` (format in `call-chain.md`). |
| Data Flow | The type chain (request, command, entity, SQL parameters, rows, response). Then `Stage \| Type \| Produced by \| Transformation \| Evidence \| Label`. Subsection **State mutations**: the state diagram, then `Field \| From \| To \| Set by (step, symbol) \| Persisted at (step) \| Label`. |
| Branches | Each decision (`B1`, `B2` ...): its diagram, then `Arm \| Guard \| Executes \| Outcome \| Reachable on this path \| Evidence \| Label`. Then **Paths**: `Path \| Decisions taken \| Outcome (response, state) \| Label` for the happy path and each distinct alternative. |
| Error Paths | `Error \| Origin (step) \| Propagation \| Handling \| State left behind \| Response \| Label`, then an error-path chain for each important error. |
| Async / Concurrent Execution | Concurrent branches (`PAR` and `JOIN` with their semantics), spawned work, and async boundaries. For each: spawn point, what runs, context passed, who waits (or nobody), and what happens to its errors and panics. Then each separate sequence (`Sequence A` ...) with its entry point and steps, when it is in the repository. |
| Database / Transaction Sequence | `# \| Step \| Operation \| Table \| Statement source \| Tx \| Result check \| Evidence \| Label`, where statement source is literal SQL or an ORM call. Then each transaction as a `BEGIN ... COMMIT` block with its rollback exits and what runs outside it. |
| Return Path | The return diagram: values and their transformations, where the response is written, and everything that still executes after the response is written. |
| Evidence | Grouped by file: the file, then symbols (with line ranges if you read them), then the step IDs each one proves. |
| Confirmed / Inferred / Unknown | Three lists, one item per line, each with its evidence or reasoning. Subsection **Not on this path**: code that looks related but does not execute (not wired, never called, unreachable arm, disabled flag), with the reason. Subsection **Documentation drift**: `Claim \| Source \| What the code does \| Evidence`. |
| Open Questions | Only the `UNKNOWN`s that matter: `Question \| Why it matters \| Where the answer likely lives`. |
| Appendix: Sequence Model | The model from section 5, in a `text` code block. |

## 7. HTML artifact: rules and skeleton

Rules:

- The HTML has the same sections, in the same order, with the same claims as the Markdown. Use
  these section ids: `overview`, `entry-point`, `main-sequence`, `call-chain`, `data-flow`,
  `branches`, `error-paths`, `async`, `database`, `return-path`, `evidence`, `confidence`,
  `open-questions`, `sequence-model`.
- Every text diagram from the Markdown appears verbatim in a `<pre class="diagram">`.
- The Main Sequence also gets a lifeline diagram (`div.seq`, below). Each arrow in it shows a step
  from the Markdown step table with its step ID. The HTML adds no steps and drops none. Group
  steps into `frame` elements for `alt` (branches), `loop`, `par`, and `tx`, and place a
  `boundary` row at every async boundary.
- The Function Call Chain also gets a call tree (`ul.tree`). Use `<details>` for Level 3
  expansions.
- Labels are badges: `<span class="label confirmed">CONFIRMED</span>`, and likewise `inferred`
  and `unknown`.
- The file is self-contained: one inline `<style>`. No `<link>` or `<script src>`, no web fonts,
  no CDN, no `<img>`, `<svg>`, or `<canvas>`. It opens directly from disk. JavaScript is not
  needed. If you add a script, it must be inline and small, and all content must be visible
  without it.
- Escape `<`, `>`, and `&` in code, diagrams, and table cells.

**Lifeline geometry.** Set `--n` on `.seq` to the number of participants, which is also the number
of `.p` headers. The grid has `2n` columns. For participant `i` (counting from 1):

| Element | `grid-column` |
|---|---|
| header `.p`, or a `.note` on participant `i` | `2i-1 / 2i+1` |
| message `.m` between participants `a` and `b` | `2*min(a,b) / 2*max(a,b)`. Add class `r` when the arrow points right (a < b) and `l` when it points left. |

Message classes: none for a call, `ret` for a return, `async` for a handoff that nobody waits for,
`err` for an error exit. Participant classes: `actor`, none for code, `db`, `ext`, `broker`,
`worker`. Put one message or note in each `.lane`. Keep participants to 8 or fewer. With more, fold
in-process helpers into notes on their caller.

Skeleton (fill in every section; repeat the patterns as needed):

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Process Refund Sequence Flow - MDW sequence-flow</title>
<style>
  :root {
    --bg: #fafaf9; --fg: #1d1f21; --muted: #5f6368; --line: #d9dcdf; --card: #fff; --code: #f2f3f4;
    --accent: #2457a6; --ok: #1e6b37; --ok-bg: #e3f2e7; --inf: #7a5200; --inf-bg: #fbf0d0;
    --unk: #9b1c22; --unk-bg: #fbe4e4; --async: #7b3fa0; --db: #0f6e6e; --ext: #8a4b0f; --err: #b3261e;
    --mono: ui-monospace, SFMono-Regular, Menlo, Consolas, "Liberation Mono", monospace;
    --sans: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", sans-serif;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --bg: #151719; --fg: #e3e5e8; --muted: #9aa0a6; --line: #33373c; --card: #1c1f23; --code: #22262a;
      --accent: #8ab4f8; --ok: #81c995; --ok-bg: #1d3124; --inf: #f5cf66; --inf-bg: #362d17;
      --unk: #f28b82; --unk-bg: #3a1d1d; --async: #c58af9; --db: #6fd3d3; --ext: #f0a868; --err: #f28b82;
    }
  }
  * { box-sizing: border-box; }
  body { margin: 0; background: var(--bg); color: var(--fg); font: 15px/1.55 var(--sans); }
  main { max-width: 1180px; margin: 0 auto; padding: 24px 16px 64px; }
  h1 { font-size: 1.6rem; margin: 0 0 8px; }
  h2 { font-size: 1.2rem; margin: 40px 0 12px; padding-top: 12px; border-top: 1px solid var(--line); }
  h3 { font-size: 1rem; margin: 20px 0 8px; }
  a { color: var(--accent); }
  .meta { display: flex; flex-wrap: wrap; gap: 4px 20px; color: var(--muted); font-size: .85rem; }
  nav.toc ol { columns: 2; margin: 12px 0 0; padding-left: 20px; font-size: .9rem; }
  code, pre { font-family: var(--mono); font-size: .85em; }
  code { background: var(--code); padding: 1px 4px; border-radius: 3px; }
  pre { background: var(--code); border: 1px solid var(--line); border-radius: 6px; padding: 12px;
        overflow-x: auto; line-height: 1.35; }
  pre code { background: none; padding: 0; }
  .table-wrap { overflow-x: auto; margin: 8px 0 16px; }
  table { border-collapse: collapse; width: 100%; font-size: .87rem; }
  th, td { border: 1px solid var(--line); padding: 6px 8px; text-align: left; vertical-align: top; }
  th { background: var(--code); font-weight: 600; white-space: nowrap; }
  .label { display: inline-block; font: 600 .7rem/1.7 var(--mono); padding: 0 6px; border-radius: 3px;
           white-space: nowrap; }
  .label.confirmed { color: var(--ok); background: var(--ok-bg); }
  .label.inferred { color: var(--inf); background: var(--inf-bg); }
  .label.unknown { color: var(--unk); background: var(--unk-bg); }
  .findings li { margin-bottom: 4px; }

  /* Lifeline diagram: n participants on 2n equal columns; participant i's lifeline is grid line 2i. */
  .seq-wrap { overflow-x: auto; margin: 8px 0 16px; }
  .seq { --n: 6; min-width: calc(var(--n) * 140px); font: .78rem/1.35 var(--mono); }
  .seq .lane { display: grid; grid-template-columns: repeat(calc(var(--n) * 2), minmax(0, 1fr));
               align-items: end; min-height: 34px; position: relative; z-index: 1; }
  .seq .heads { align-items: stretch; }
  .seq .p { margin: 0 6px; padding: 6px 4px; text-align: center; font: 600 .8rem/1.3 var(--sans);
            background: var(--card); border: 1px solid var(--line); border-top: 3px solid var(--accent);
            border-radius: 4px; overflow-wrap: anywhere; }
  .seq .p small { display: block; font: .68rem/1.3 var(--mono); color: var(--muted); }
  .seq .p.actor { border-top-color: var(--muted); }
  .seq .p.db { border-top-color: var(--db); }
  .seq .p.ext { border-top-color: var(--ext); }
  .seq .p.broker, .seq .p.worker { border-top-color: var(--async); }
  .seq .body { position: relative; padding: 4px 0 8px; }
  .seq .body::before { content: ""; position: absolute; inset: 0; z-index: 0; pointer-events: none;
    background: linear-gradient(to right, transparent calc(50% - 1px), var(--line) calc(50% - 1px),
                var(--line) calc(50% + 1px), transparent calc(50% + 1px)) 0 0 / calc(100% / var(--n)) 100% repeat-x; }
  .seq .m { position: relative; padding: 0 6px 4px; text-align: center; color: var(--fg);
            border-bottom: 2px solid var(--fg); }
  .seq .m::after { content: ""; position: absolute; bottom: -6px; border: 5px solid transparent; }
  .seq .m.r::after { right: -1px; border-left: 8px solid var(--fg); border-right: 0; }
  .seq .m.l::after { left: -1px; border-right: 8px solid var(--fg); border-left: 0; }
  .seq .m.ret { color: var(--muted); border-bottom: 2px dashed var(--muted); }
  .seq .m.ret.r::after { border-left-color: var(--muted); }
  .seq .m.ret.l::after { border-right-color: var(--muted); }
  .seq .m.async { color: var(--async); border-bottom: 2px dotted var(--async); }
  .seq .m.async.r::after { border-left-color: var(--async); }
  .seq .m.async.l::after { border-right-color: var(--async); }
  .seq .m.err { color: var(--err); border-bottom-color: var(--err); }
  .seq .m.err.r::after { border-left-color: var(--err); }
  .seq .m.err.l::after { border-right-color: var(--err); }
  .seq .id { color: var(--muted); font-weight: 600; margin-right: 4px; }
  .seq .note { margin: 2px 4px; padding: 3px 6px; text-align: center; background: var(--card);
               border: 1px solid var(--line); border-left: 3px solid var(--accent); border-radius: 3px; }
  .seq .note.state { border-left-color: var(--ok); }
  .seq .note.err { border-left-color: var(--err); color: var(--err); }
  .seq .frame { position: relative; margin: 6px 0; padding: 22px 0 4px; }
  .seq .frame::before { content: ""; position: absolute; inset: 0 calc(var(--d, 0) * 6px); z-index: 0;
                        border: 1px solid var(--muted); border-radius: 4px; pointer-events: none; }
  .seq .frame > .tag { position: absolute; top: 0; left: calc(var(--d, 0) * 6px); z-index: 2;
                       padding: 1px 8px; font-weight: 600; background: var(--muted); color: var(--bg);
                       border-radius: 4px 0 4px 0; }
  .seq .frame.tx::before { border-color: var(--db); border-width: 2px; }
  .seq .frame.tx > .tag { background: var(--db); }
  .seq .frame.par::before { border-style: double; border-width: 3px; }
  .seq .guard { position: relative; z-index: 2; margin: 2px calc(var(--d, 0) * 6px) -14px; padding: 0 8px;
                line-height: 16px; color: var(--muted); font-weight: 600; }
  .seq .guard.else { border-top: 1px dashed var(--muted); }
  .seq .boundary { position: relative; z-index: 2; margin: 10px 0; padding: 6px 10px; text-align: center;
                   font-weight: 600; color: var(--async); background: var(--bg);
                   border-top: 2px dashed var(--async); border-bottom: 2px dashed var(--async); }
  .legend { display: flex; flex-wrap: wrap; gap: 4px 18px; font: .75rem/1.6 var(--mono); color: var(--muted); }
  .legend i { display: inline-block; width: 28px; vertical-align: middle; margin-right: 6px;
              border-bottom: 2px solid var(--fg); }
  .legend i.ret { border-bottom: 2px dashed var(--muted); }
  .legend i.async { border-bottom: 2px dotted var(--async); }
  .legend i.err { border-bottom-color: var(--err); }

  /* Call tree: nesting = callee under caller; top to bottom = execution order. */
  ul.tree, ul.tree ul { list-style: none; margin: 0; padding-left: 20px; }
  ul.tree { padding-left: 0; font: .82rem/1.5 var(--mono); }
  ul.tree li { position: relative; padding: 2px 0 2px 14px; border-left: 1px solid var(--line); }
  ul.tree li:last-child { border-left-color: transparent; }
  ul.tree li::before { content: ""; position: absolute; left: 0; top: 0; width: 10px; height: 13px;
                       border-left: 1px solid var(--line); border-bottom: 1px solid var(--line); }
  ul.tree .id { color: var(--muted); margin-right: 6px; }
  ul.tree .where { color: var(--muted); font-size: .9em; margin-left: 8px; }
  ul.tree .k { display: inline-block; padding: 0 5px; margin-right: 6px; border-radius: 3px;
               font-weight: 600; font-size: .85em; border: 1px solid currentColor; }
  ul.tree .k.db { color: var(--db); }
  ul.tree .k.ext { color: var(--ext); }
  ul.tree .k.async { color: var(--async); }
  ul.tree .k.err { color: var(--err); }
  ul.tree .k.ctl { color: var(--muted); }
  ul.tree details > summary { cursor: pointer; }
  @media print { body { background: #fff; } pre, .seq, table { break-inside: avoid; } }
</style>
</head>
<body>
<main>
  <header>
    <h1>Process Refund Sequence Flow</h1>
    <div class="meta">
      <span>Repository: <code>billing</code></span>
      <span>Commit: <code>9c1e0d2</code></span>
      <span>Analyzed: 2026-09-28</span>
      <span>Level: 2</span>
      <span>Generated by MDW sequence-flow</span>
    </div>
    <nav class="toc"><ol>
      <li><a href="#overview">Overview</a></li>
      <li><a href="#entry-point">Entry Point</a></li>
      <!-- one item per included section, in Markdown order -->
    </ol></nav>
  </header>

  <section id="overview">
    <h2>1. Overview</h2>
    <p>Summary paragraph.</p>
    <h3>Key findings</h3>
    <ul class="findings">
      <li><span class="label confirmed">CONFIRMED</span> The PSP refund (6a) happens before T1;
        if T1 fails, the money is returned but no refund row exists.</li>
    </ul>
  </section>

  <section id="main-sequence">
    <h2>3. Main Sequence</h2>
    <pre class="diagram">[Client]
   |  POST /refunds
   v
RequireAuth()                               x--&gt; 401 unauthorized</pre>
    <div class="legend"><span><i></i>call</span><span><i class="ret"></i>return</span>
      <span><i class="async"></i>not awaited</span><span><i class="err"></i>error exit</span></div>
    <div class="seq-wrap">
      <div class="seq" style="--n: 5">
        <div class="lane heads">
          <div class="p actor" style="grid-column: 1 / 3">Client</div>
          <div class="p" style="grid-column: 3 / 5">RefundHandler<small>internal/http/refund.go</small></div>
          <div class="p" style="grid-column: 5 / 7">RefundService<small>internal/refund/service.go</small></div>
          <div class="p db" style="grid-column: 7 / 9">PostgreSQL<small>DATABASE_URL</small></div>
          <div class="p broker" style="grid-column: 9 / 11">RabbitMQ<small>exchange billing</small></div>
        </div>
        <div class="body">
          <div class="lane"><div class="m r" style="grid-column: 2 / 4"><span class="id">1</span>POST /refunds</div></div>
          <div class="lane"><div class="m err l" style="grid-column: 2 / 4"><span class="id">2x</span>400 invalid_json</div></div>
          <div class="lane"><div class="m r" style="grid-column: 4 / 6"><span class="id">3</span>ProcessRefund(ctx, cmd)</div></div>
          <div class="frame" style="--d: 0"><span class="tag">alt p.Method</span>
            <div class="guard">[CARD]</div>
            <div class="lane"><div class="note" style="grid-column: 5 / 7"><span class="id">6a</span>PSPClient.Refund()</div></div>
            <div class="guard else">[default]</div>
            <div class="lane"><div class="m err l" style="grid-column: 4 / 6"><span class="id">6x</span>ErrUnsupportedMethod</div></div>
          </div>
          <div class="lane"><div class="note state" style="grid-column: 5 / 7"><span class="id">7</span>REQUESTED &rarr; COMPLETED</div></div>
          <div class="frame tx" style="--d: 0"><span class="tag">tx T1</span>
            <div class="lane"><div class="m r" style="grid-column: 6 / 8"><span class="id">8</span>BEGIN; INSERT refunds</div></div>
            <div class="frame" style="--d: 1"><span class="tag">loop each line</span>
              <div class="lane"><div class="m r" style="grid-column: 6 / 8"><span class="id">8.1</span>INSERT refund_lines</div></div>
            </div>
            <div class="lane"><div class="m r" style="grid-column: 6 / 8"><span class="id">8.2</span>COMMIT</div></div>
          </div>
          <div class="lane"><div class="m r" style="grid-column: 6 / 10"><span class="id">9</span>publish refund.completed</div></div>
          <div class="lane"><div class="m ret l" style="grid-column: 2 / 4"><span class="id">11</span>201 Created</div></div>
          <div class="boundary">ASYNC BOUNDARY: refund.completed is delivered later, to cmd/notifier (Sequence A)</div>
        </div>
      </div>
    </div>
    <div class="table-wrap"><table>
      <thead><tr><th>#</th><th>Caller</th><th>Callee</th><th>Call / action</th><th>Kind</th><th>Evidence</th><th>Label</th></tr></thead>
      <tbody><tr><td>3</td><td>RefundHandler.Create</td><td>RefundService.ProcessRefund</td><td>ProcessRefund(ctx, cmd)</td>
        <td>CALL</td><td><code>internal/http/refund.go Create()</code></td><td><span class="label confirmed">CONFIRMED</span></td></tr></tbody>
    </table></div>
  </section>

  <section id="call-chain">
    <h2>4. Function Call Chain</h2>
    <ul class="tree">
      <li><span class="id">2</span><code>RefundHandler.Create()</code><span class="where">internal/http/refund.go</span>
        <span class="k err">x 400</span>
        <ul>
          <li><span class="id">3</span><code>RefundService.ProcessRefund()</code><span class="where">internal/refund/service.go</span>
            <ul>
              <li><span class="id">4</span><code>RefundRepository.FindPayment()</code> <span class="k db">SELECT payments</span></li>
              <li><span class="id">6</span><span class="k ctl">SWITCH</span>p.Method
                <ul>
                  <li><span class="id">6a</span>[CARD] <code>PSPClient.Refund()</code> <span class="k ext">HTTP POST /v1/refunds</span></li>
                  <li><span class="id">6x</span>[default] <span class="k err">x ErrUnsupportedMethod</span></li>
                </ul></li>
              <li><details open><summary><span class="id">8</span><code>RefundRepository.Save()</code> <span class="k ctl">TX T1</span></summary>
                <ul>
                  <li><span class="id">8.1</span><span class="k ctl">LOOP</span><span class="k db">INSERT refund_lines</span></li>
                </ul></details></li>
              <li><span class="id">9</span><code>Publisher.Publish()</code> <span class="k async">~&gt; refund.completed</span></li>
            </ul></li>
        </ul></li>
    </ul>
  </section>

  <section id="error-paths">
    <h2>7. Error Paths</h2>
    <div class="table-wrap"><table>
      <thead><tr><th>Error</th><th>Origin</th><th>Propagation</th><th>Handling</th><th>State left behind</th><th>Response</th><th>Label</th></tr></thead>
      <tbody><tr><td>PSP declined</td><td>6a</td><td>returned unchanged</td><td>statusFor() maps to 402</td>
        <td>nothing written</td><td>402 payment_declined</td><td><span class="label confirmed">CONFIRMED</span></td></tr></tbody>
    </table></div>
    <pre class="diagram">E2  PSP declines the refund
  origin    6a  PSPClient.Refund() maps HTTP 402 to ErrPSPDeclined</pre>
  </section>

  <section id="sequence-model">
    <h2>Appendix: Sequence Model</h2>
    <pre>SEQUENCE: Process Refund
SOURCE: billing @ 9c1e0d2 (2026-09-28)</pre>
  </section>
</main>
</body>
</html>
```
