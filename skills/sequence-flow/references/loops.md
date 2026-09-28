# Loops

A loop repeats steps, so it multiplies side effects, and a failure in iteration *k* leaves
iterations *1..k-1* behind. A sequence that draws a loop body once, with no exits and no bound,
hides where partial work, per-item round trips, and ordering problems come from.

## Contents

1. Kinds of loops
2. What to record
3. Exits
4. Partial failure
5. Order of iteration
6. Loops that spawn work
7. Retry and polling loops
8. Representation

## 1. Kinds of loops

| Kind | Code shapes | Sequence meaning |
|---|---|---|
| Collection iteration | `for _, x := range xs`, `for x in xs`, `for (const x of xs)`, `xs.forEach`, `xs.map` | the body runs once per element |
| Condition loop | `for cond {}`, `while`, `do ... while` | the bound depends on state; find what changes it |
| Pagination or batching | loop until `next == ""`, `LIMIT/OFFSET`, cursor | repeated calls to a store or API; the total count is data-dependent |
| Retry | loop around a call, with sleep or backoff | the same side effect attempted several times (section 7) |
| Polling | loop with sleep until a condition holds | repeated reads; a timeout or cancellation exit |
| Consumer or worker loop | `for msg := range ch`, `for d := range deliveries`, `while True: poll()` | the entry of a worker's sequence: each iteration is one run of the handler sequence |
| Recursion | a function calling itself or a cycle of functions | a loop with the call stack as its state; find the base case |

**Not a loop in the sequence:** a single bulk statement (`INSERT ... VALUES (...), (...)`,
`COPY`, `CopyFrom`, `bulk_create`, `insertMany`, `WHERE id = ANY($1)`) is one round trip. Draw it
as one step, even if code before it loops to build the parameters.

## 2. What to record

```text
L1  <file> <Symbol>: LOOP <over what>
    Source       where the collection or condition comes from (step ID)
    Bound        its maximum, and the evidence (a validation limit, a page size), or unbounded, or UNKNOWN
    Body         the steps inside, with IDs (8.1, 8.2)
    Per iteration  side effects: queries, remote calls, writes, messages
    Exits        normal end, break, continue, return, error, cancellation
    On failure   what iterations 1..k-1 left behind (section 4)
```

Record the bound when the body leaves the process. "One `INSERT` per line, at most 20 lines
(`validate()` enforces the limit)" is useful. "One HTTP call per SKU, unbounded" is a finding.

## 3. Exits

| Statement | Effect |
|---|---|
| `break` | leaves the innermost loop, or the labeled loop in Go and Java; execution continues after the loop |
| `continue` | skips to the next iteration. Record it when it silently skips items ("invalid lines are skipped, not rejected"). |
| `return` inside the loop | leaves the whole function, and runs deferred and `finally` code on the way out |
| error or exception inside the body | follows the error path; say whether it stops the loop (first error wins) or is collected (all items attempted) |
| context cancellation or timeout checked in the loop | leaves the loop part way through |

Traps that change the meaning of an exit:

- **JavaScript `forEach`, `map`, and similar.** The body is a callback. `return` inside it skips
  only that element; it cannot stop the loop or return from the enclosing function. `break` is
  not allowed.
- **`await` inside `forEach`.** `xs.forEach(async x => await save(x))` starts every `save`
  without waiting. The enclosing function continues immediately, and rejections are unhandled.
  This is an async boundary (`async-boundary.md`), not a sequential loop. `for (const x of xs) {
  await save(x) }` is sequential. `await Promise.all(xs.map(save))` is concurrent and waits for
  all of them.
- **Python generators and comprehensions.** A generator body runs only as it is consumed. A
  comprehension used only for its side effects still runs eagerly, once per element.

## 4. Partial failure

For every loop whose body writes or calls out, answer: if iteration *k* fails, what happens to
the effects of iterations *1..k-1*?

| Where the body's effects go | What remains after a failure at iteration k |
|---|---|
| Statements inside one database transaction that rolls back on this error | nothing; the rollback undoes them. Say which rollback: `defer`, explicit, framework. |
| Separate autocommit statements | rows for items 1..k-1 remain |
| Remote calls or published messages | calls 1..k-1 happened and are not undone; say whether anything compensates |
| Errors collected and the loop continues | every item is attempted; say how the collected errors are reported |

You read the loop and the transaction, so which statements are inside the loop is `CONFIRMED`.
A consequence such as "the customer is emailed twice on retry" is `INFERRED`, with the
reasoning.

## 5. Order of iteration

Record order only when it affects the outcome: which item fails first, which write wins, the
order in which messages are published.

- Go `range` over a **map** has a randomized order. The order of side effects in that loop is
  not deterministic.
- Slices, arrays, and lists iterate in order. JavaScript `Map` and `Set` iterate in insertion
  order. Plain object keys follow property-order rules: integer-like keys first.
- A query without `ORDER BY` returns rows in no guaranteed order.
- Concurrent iterations (section 6) complete in no guaranteed order.

## 6. Loops that spawn work

A loop that starts a goroutine, thread, task, or promise per element is a fan-out. Record:

- **Join**: `WaitGroup.Wait`, `errgroup.Wait`, `Promise.all`, `asyncio.gather`,
  `CompletableFuture.allOf(...).join()`, or none. With no join, the loop's work is async
  (`async-boundary.md`).
- **Limit**: `errgroup.SetLimit`, a semaphore, a worker pool, `p-limit`, or unbounded.
- **Error behavior**: first error cancels the rest, all run to completion, or errors are dropped.
- **Loop variable capture (Go before 1.22).** Closures capture the shared loop variable, so every
  goroutine may see the last element. Check the `go` directive in `go.mod`. From 1.22 on, each
  iteration has its own variable. Flag this only when the code captures the variable.

## 7. Retry and polling loops

Document a retry only when the code has one. For each retry, record where it is, the maximum
number of attempts, the backoff, which errors trigger it, which errors stop it, and whether the
repeated call is idempotent. Retries can also live in client libraries, annotations, broker
redelivery, or infrastructure; what the repository cannot show is `UNKNOWN`. A comment or README that says "retries" is not a retry.

A polling loop needs its exit condition and its timeout. With neither, it can run forever.

## 8. Representation

```text
8   RefundRepository.Save()   (inside T1)
    |
    +-- LOOP L1 for each l in r.Lines          bound: 1..20, validate()
    |     8.1  +--> INSERT refund_lines
    |          x--> error: return err -> DEFER tx.Rollback() undoes 8 and lines 1..k-1
    |
    +--> COMMIT
```

In the HTML lifeline diagram, wrap the body steps in a `frame` with the tag `loop <over what>`.
In the call tree, mark the step `LOOP` and indent the body under it.
