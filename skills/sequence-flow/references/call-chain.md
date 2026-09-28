# Call Chain

The call chain is the list of calls that actually execute, each resolved to the code that runs.
Getting the callee right matters more than anything else in a sequence: a call attributed to the
wrong implementation makes every step below it fiction.

## Contents

1. What one step records
2. Resolving the callee
3. Hidden calls
4. Function records
5. Granularity: Level 1, 2, 3
6. Where the chain stops

## 1. What one step records

```text
<id>  <caller Symbol> -> <callee Symbol>  <purpose>   <file of callee>   <LABEL>
```

- **Caller** and **callee** are code symbols (`RefundService.ProcessRefund`), not layer names
  ("the service").
- **Purpose** is what this call does for this operation, in a few words ("load payment",
  "charge card"), not the function's general description.
- Record calls in the order the caller's code makes them. If two calls can run in either order,
  do not invent one; see `async-boundary.md`.

## 2. Resolving the callee

A call site names a symbol. The code that runs may be somewhere else. Resolve every call on the
path before you expand it.

| Call site | What runs | How to confirm |
|---|---|---|
| Plain function `validate(cmd)` | that function | read it: `CONFIRMED` |
| Method on a concrete type | that type's method, or the promoted method of an embedded type (Go), or the most-derived override (class inheritance) | find the receiver's static type. For inheritance, check the subclasses that can be instantiated on this path. |
| Method on an interface or abstract type | the implementation that was constructed and injected into this process | find the constructor call in `main`, bootstrap, or the DI container (wire, fx, dig, Spring `@Bean`/`@Primary`/`@Profile`, Nest providers, Guice, .NET `services.Add*`). One wired implementation: `CONFIRMED`. Chosen by runtime config: `INFERRED`, citing the config key. |
| Wrapper around a dependency | the wrapper first (logging, metrics, caching, retry, circuit breaker, tracing), then the wrapped implementation | constructor chains such as `NewCachedRepo(NewSQLRepo(db))`. A wrapper that changes behavior (cache hit returns early, retry repeats the call) is a step. A wrapper that only observes (metrics, tracing) is a one-line note. |
| Function value, callback, closure | whatever was assigned to that variable or field | find the assignment. Callbacks run when the callee invokes them, not where they are defined; place them in the sequence at the point of invocation. |
| Registry or strategy map (`handlers[evt.Type]`, `strategies.get(method)`) | the entry for the key that this input produces | find where the map is populated, then which key this path produces. If the key comes from input, each key is a branch (`branching.md`). |
| Event bus or mediator dispatch | every handler registered for that event or command type | find the registrations. Check whether dispatch is synchronous (`async-boundary.md`). |
| Generic function | the generic body, specialized for this path's type argument | read the body. A constraint's method is resolved like an interface method. |
| Reflection, dynamic import, `eval`, plugin loading, `method_missing`, `__getattr__` | decided at runtime | `INFERRED` if the name is visible as a literal; otherwise `UNKNOWN`. |
| Generated client (gRPC stub, OpenAPI client, ORM query builder) | a remote call or a query | stop at the stub. Record the RPC, path, or query, not the generated internals. |

Never pick an implementation because its name matches, because it is the only one you found by
search, or because it is in the same package. Confirm it is the one wired into this process.
`StubPaymentClient`, `FakeMailer`, and `InMemoryRepo` often sit next to the real ones.

## 3. Hidden calls

These execute code that does not appear at the call site. Check the ones your stack uses.

| Mechanism | Examples | What to record |
|---|---|---|
| Proxies and aspects | Spring `@Transactional`, `@Async`, `@Cacheable`, `@Retryable`, AOP advice, .NET filters | the behavior the proxy adds before and after the method. Self-invocation (`this.method()`) bypasses a Spring proxy, so the annotation has no effect on that call. |
| Decorators | Python `@decorator`, TypeScript decorators, NestJS guards, pipes, and interceptors | the wrapper runs first, and it decides whether the wrapped function runs at all |
| Context managers and disposables | Python `with` (`__enter__`/`__exit__`), C# `using`, Java try-with-resources | enter at the start of the block, exit at the end, including on exceptions |
| Deferred and cleanup code | Go `defer`, `finally`, Ruby `ensure` | runs at scope exit, in LIFO order for Go; see `control-flow.md` |
| ORM hooks and callbacks | GORM `BeforeCreate`/`AfterSave`, JPA `@PrePersist`/`@PostUpdate`, Rails `before_save`/`after_commit`, Django signals, Sequelize and TypeORM hooks | extra calls, often including I/O, that run inside save or delete |
| Lazy loading | ORM relations loaded on first access | a query at the line that reads the field, not at the load call |
| Property getters and setters, operator overloads, `toString`/`__str__`, `MarshalJSON`/`toJSON` | computed properties, custom serializers | a call only when it does work that matters (I/O, validation, a state change) |
| Init and static initialization | Go `init()`, static blocks, module top-level code | runs once at startup; mention it only if the path depends on what it set up |
| Database triggers | `CREATE TRIGGER` in migrations | a side effect inside the database statement, `CONFIRMED` from the migration |

## 4. Function records

Write one record for each **meaningful** function on the path: one that decides something,
writes state, leaves the process, transforms the result, or can fail in a way the caller
handles.

```text
RefundService.ProcessRefund()                       internal/refund/service.go   CONFIRMED
  Input         ctx, refund.Command{PaymentID, AmountCents, Lines}
  Output        *refund.Refund, error
  Calls         4 RefundRepository.FindPayment()
                6a PSPClient.Refund()        [CARD]
                6b Ledger.Credit()           [STORE_CREDIT]
                7 Refund.Complete()
                8 RefundRepository.Save()
                9 Publisher.Publish()
  Side effects  PSP refund (6a) or ledger credit (6b); refunds and refund_lines rows (T1);
                refund.completed message
  Errors        ErrNotFound (from 4), ErrNotRefundable (5), ErrUnsupportedMethod (6x),
                ErrPSPDeclined and ErrPSPUnavailable (from 6a), T1 errors (from 8).
                Publish errors are logged and not returned.
```

- **Calls** lists call steps in order, with their step IDs and the arm or loop they belong to.
- **Errors** lists what the function returns or throws, and under which condition. It also lists
  errors it receives and does **not** pass on (swallowed, logged, replaced).
- Skip fields that are empty. Do not write records for pass-through functions. Name them in the
  call tree and move on.

## 5. Granularity: Level 1, 2, 3

| Level | Shows | Use for |
|---|---|---|
| 1. Business sequence | components: `Handler -> Service -> Repository -> Database` | the Overview summary, and a very long flow where you need a map first |
| 2. Function sequence | the functions and methods that hold control: `RefundHandler.Create() -> RefundService.ProcessRefund() -> RefundRepository.Save() -> tx.Exec()` | **the default** for the main sequence and the call tree |
| 3. Internal detail | the internals of one function: `ProcessRefund() -> validate() -> remaining() -> newRefund() -> repo.Save()` | only the functions whose internals change the outcome |

**Expand a function to Level 3 when** its internals decide something the reader needs: which
branch is taken, what is written, what is returned, which error comes out, or whether a later
step runs at all. Expand that function only, not its siblings.

**Do not expand** standard-library and trivial helpers (`strings.TrimSpace`, `time.Now`, `len`,
`fmt.Sprintf`, getters), logging, metrics, or tracing. Expand one only when the outcome depends
on its specific behavior. For example, `time.Now()` compared to an expiry makes the comparison a
decision; the decision is the step, not `time.Now`.

**Collapse repetition.** Five mapper calls in a row become one line, "maps request fields to
`refund.Command`", with the mapper named once.

State the level in the Overview and the model (`LEVEL: 2; level 3 in ProcessRefund`).

## 6. Where the chain stops

Stop expanding a call when it reaches:

- **a database, cache, or broker**: record the operation (`execution-tracing.md`) and go no
  further
- **an external system**: record the client function, protocol, target, request, and error
  handling, then stop
- **framework or library code**: record its behavior as an `INFERRED` default, with the reason
- **code in another repository**: record the contract (URL and path, RPC, topic, payload type)
  and mark the receiver's internals `UNKNOWN`
- **an async boundary**: the continuation is a separate sequence (`async-boundary.md`)
- **work outside the operation's scope**: generic logging, metrics, health checks
