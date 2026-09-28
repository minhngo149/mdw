# Control Flow

Control flow decides which statement runs next. A sequence that shows the calls but not the
decisions between them tells the reader what *can* happen, not what *does* happen. This reference
covers execution order, guards, error propagation, cleanup code, panics, and the return path.
Branches with several arms are covered in `branching.md`, loops in `loops.md`, and concurrency in
`async-boundary.md`.

## Contents

1. Execution order is not always source order
2. Guard clauses and early returns
3. Error paths
4. Exceptions
5. Deferred and cleanup code
6. Panics and crashes
7. Return path
8. Work after the response

## 1. Execution order is not always source order

Statements in one function run top to bottom, except in these cases:

| Construct | When it actually runs |
|---|---|
| Go `defer f(x)`, `finally`, `ensure`, `using`, `with` exit | at scope exit, after the `return` value is computed. Go runs defers in LIFO order. Go evaluates `defer` **arguments** when the `defer` statement runs, not at exit. |
| Short-circuit `a && b()`, `a \|\| b()`, `a ?? b()`, `a?.b()` | `b()` runs only when `a` does not decide the result |
| Ternaries and conditional expressions | only the chosen side runs |
| Callbacks, closures, and handlers passed to a function | when the receiver invokes them, which may be later, several times, or never |
| Lazy sequences: generators, iterators, Java streams, LINQ, Rx and Reactor pipelines, Django querysets | at the terminal operation (`collect`, `toList`, `subscribe`, iteration, `len()`), not where the pipeline is built |
| `await` | the caller suspends and resumes at this line. For this caller the order is still sequential. |
| Middleware `next()` | code before `next()` runs on the way in, code after it runs on the way out, in reverse order of registration |
| Go `select`, JS `Promise.race` | whichever case is ready first; the order is nondeterministic |

When the order matters to the outcome, show it the way it executes and add a one-line note, for
example "the deferred `tx.Rollback()` runs after `return err`".

## 2. Guard clauses and early returns

A guard is a check that leaves the function before the rest of it runs. Every guard on the path
is a step, because it decides whether everything after it runs.

```text
RefundService.ProcessRefund()
    |
    +-- payment found?            x--> NO: return ErrNotFound
    +-- p.Status == SETTLED?      x--> NO: return ErrNotRefundable
    +-- amount <= remaining(p)?   x--> NO: return ErrAmountTooLarge
    |
    +--> continue to step 6
```

- Write the condition as the code states it, or as a faithful paraphrase. Do not rename it to
  sound like a business rule the code does not implement.
- Note the side effects that have already happened when the guard exits: nothing yet, a row
  written, a remote call made. A guard after a side effect is a partial-failure point.

## 3. Error paths

Trace every meaningful error from where it starts to what the caller finally sees. At each layer
the error is handled in exactly one of these ways:

| Layer action | Code shape | What to record |
|---|---|---|
| Return as is | `return nil, err`, rethrow | "returned unchanged" |
| Wrap | `fmt.Errorf("...: %w", err)`, `new XError(msg, { cause })`, `raise X from e` | still matchable by `errors.Is` or `instanceof` on the cause, or not (for example `%v` instead of `%w`) |
| Map | `if errors.Is(err, sql.ErrNoRows) { return ErrNotFound }` | old error to new error |
| Swallow | log and continue, `_ =`, ignored return value, empty `catch`, `.catch(() => {})` | a **key finding**: the caller sees success while the work is incomplete. Quote the evidence. |
| Compensate | refund, release, delete, or status update, then return | the compensation is a step with its own error path. If its error is swallowed, say what state remains. |
| Convert to a response | error middleware, `writeError`, `@ControllerAdvice`, exception filter, gRPC status mapping | the error-to-response table |
| Crash | `panic`, unhandled exception, `process.exit`, `log.Fatal` | see section 6 |

**The error-to-response table.** Find the single place that turns errors into responses, and
record every case it has, including the default. An error that no case matches gets the
default, often 500. Check each error the path can produce against the table: a sentinel that is
wrapped in the wrong way, or never listed, gets the default even when the author clearly meant
something else. That is a `CONFIRMED` finding when you read both the error and the table.

**Errors on the error path.** When the handling itself can fail (the compensation call, the
status update to `FAILED`, the rollback), trace that failure one more step and say what state it
leaves behind.

**Ignored results that are not errors.** Unchecked rows-affected counts, discarded decode
results, and ignored `ok` values change what happens next without raising anything. For example,
a decode error that is ignored leaves a zero value that a later check misreads. Record them when
they change the outcome.

Format each important error as a chain (the error path pattern in `sequence-format.md`):
origin, then each layer up, then the mapping, the state left behind, and the response.

## 4. Exceptions

In exception-based languages the error path is not visible at the call sites. Find it this way:

1. From the throw site, walk up the call chain to the nearest enclosing `try` whose `catch`
   matches the exception type. Every frame in between exits immediately, and its `finally`
   blocks run on the way out.
2. If nothing catches it, the framework does: Spring `@ExceptionHandler`/`@ControllerAdvice`,
   Express error middleware (the four-argument `(err, req, res, next)`), NestJS exception
   filters, FastAPI `exception_handler`, Django middleware, ASP.NET exception middleware. Record
   the mapping. With no mapping, the framework default is `INFERRED`.
3. Check what the exception does to a transaction on its way out. Spring rolls back on unchecked
   exceptions only, unless `rollbackFor` says otherwise. A `catch` that swallows the exception
   inside `@Transactional` lets the transaction commit.
4. Async code: an exception in an un-awaited promise or a background task does not reach the
   caller's `try`. It becomes an unhandled rejection, a logged error, or nothing.

## 5. Deferred and cleanup code

Record cleanup code where it runs, at scope exit, and on every exit path it affects.

| Pattern | Sequence meaning |
|---|---|
| `defer tx.Rollback()` right after `Begin` | runs at every return. After a successful `Commit` it is a no-op, so on the happy path it does nothing; on every error return it rolls back. |
| `defer mu.Unlock()`, `defer rows.Close()`, `defer resp.Body.Close()` | usually trivial. Mention only when the critical section or resource lifetime matters. |
| `defer func() { if r := recover(); r != nil { ... } }()` | turns a panic in **this goroutine** into whatever the function does next (section 6) |
| Named result modified in a `defer` (`defer func() { err = wrap(err) }()`) | the deferred code changes the returned value. Show it on the return path. |
| `finally` that returns or throws | replaces the original result or exception. This is a real behavior change; flag it. |
| Python `with lock:`, `with session.begin():` | entering takes the lock or begins the transaction; leaving releases or commits it, or rolls back on an exception |

## 6. Panics and crashes

- **Go.** `recover()` works only inside a deferred function in the **same goroutine** that
  panicked. Recovery middleware (chi or gin `Recoverer`, `grpc_recovery`) turns a handler panic
  into a 500. A panic inside a goroutine started with `go` is not covered by that middleware and
  crashes the whole process unless that goroutine recovers. You read the missing `recover`, so
  that is `CONFIRMED`; the crash is `INFERRED` from Go semantics.
- **Common panic sources worth flagging:** nil map writes, nil pointer dereference on a result
  whose error was ignored, out-of-range index, failed type assertion without `, ok`.
- **Fatal exits.** `log.Fatal`, `os.Exit`, `process.exit`, and `System.exit` skip deferred and
  `finally` code.
- **Workers.** A crash in a consumer usually means a restart, then redelivery of unacknowledged
  messages. Redelivery depends on the ack mode, so it is `INFERRED`. See `async-boundary.md`.

## 7. Return path

The return path is how control and data come back up the chain to the caller that started the
operation.

For each return on the happy path and on each distinct failure outcome, record:

- **Value**: what is returned (`*Refund`, `nil`, `error`).
- **Transformation**: where one representation becomes another (DB row, then entity, then
  response DTO, then JSON), with the function that does it. Record only transformations the code
  performs. Do not add a "domain model" layer the repository does not have.
- **Status decision**: where the status code, exit code, or ack decision is made, and what
  selects it.
- **Response write**: the exact call that sends the response (`w.WriteHeader`, `res.json`,
  `return ResponseEntity...`, `ctx.JSON`, returning a value to the framework). This is the point
  after which the caller has its answer.

## 8. Work after the response

Code that runs after the response is written is invisible to the caller. The caller has already
received its answer, so a failure here cannot change it. Look for:

- statements after `writeJSON`/`res.send`/`c.JSON` in the same handler
- deferred functions and `finally` blocks in the handler and in middleware
- middleware code after `next()` (logging, metrics, a per-request transaction commit)
- goroutines, threads, `BackgroundTasks`, `res.on('finish')`, `setImmediate`, un-awaited
  promises started by the handler (`async-boundary.md`)

Draw these below the response step, never above it. For each one, say what the client can no
longer learn: "if the search-index update fails, the client already has 201 and nothing retries it".
If a per-request transaction commits after the response is written, a commit failure is not
reported to the client. That is a key finding.
