# Async Boundaries and Concurrency

A sequence is a claim about order: this runs, then that. The claim holds only while the caller
waits. Where the caller stops waiting (a publish, a spawned goroutine, an un-awaited promise, an
enqueued job), the sequence forks. Where two calls run at the same time, it has no order at all
until something joins them. This reference covers finding those points and drawing them honestly.

## Contents

1. Three relationships between caller and callee
2. `async` is not an async boundary
3. Detecting boundaries and concurrency by stack
4. Crossing a process boundary: publish, enqueue, outbox
5. Spawned background work
6. Concurrent branches and joins
7. Shared state and locks
8. Representation

## 1. Three relationships between caller and callee

| Relationship | Definition | Evidence required |
|---|---|---|
| Synchronous | the caller waits for the callee's result, then continues | a plain call, or an `await`, `.get()`, `.join()`, or blocking receive on the result |
| Concurrent, joined | the caller starts several branches, then waits for them at a join point | a spawn **and** a join (`Wait`, `await Promise.all`, `gather`, `allOf().join()`) |
| Async boundary | the caller continues without the callee's result; the callee runs later or elsewhere | a spawn or handoff **and** no join on this path |

Everything after an async boundary is a **separate sequence**, with its own entry point, its own
step IDs (`A1`, `B1`), and its own failure handling. The caller's sequence continues with the
next statement after the handoff. It never continues with the callee's steps.

## 2. `async` is not an async boundary

- An **awaited** call to an `async` function is synchronous for the caller: it suspends, then
  resumes with the result. Draw it as a normal call.
- A call to an `async` function **without** `await` (or `.then`, or a later `await` on the
  stored promise) is an async boundary. Its errors do not reach the caller's `try`.
- In-process event mechanisms are often synchronous: Node `EventEmitter.emit`, Spring
  `ApplicationEventPublisher` (unless the listener is `@Async`), MediatR `Publish`, Django
  signals, Rails `ActiveSupport::Notifications`. Their listeners run in the caller's thread,
  before `emit` returns. Draw them as calls. Check the dispatcher every time.
- An **unbuffered** Go channel send blocks until a receiver takes the value. That is a sync
  handoff. A buffered send returns immediately while there is room.

## 3. Detecting boundaries and concurrency by stack

| Stack | Async boundary (nobody waits) | Concurrent + join | Synchronization |
|---|---|---|---|
| Go | `go f()` with no join; `time.AfterFunc`; a send to a worker channel | `sync.WaitGroup` + `Wait`; `errgroup.Go` + `Wait`; fan-in over a results channel | `sync.Mutex`/`RWMutex`, `sync.Once`, `atomic`, channels, `select` |
| JS / TS | un-awaited promise; `setTimeout`, `setImmediate`, `queueMicrotask`, `process.nextTick`; `forEach(async ...)`; `res.on('finish')` | `await Promise.all`, `allSettled`, `race`, `any` | the single-threaded event loop; `worker_threads` and `Atomics` |
| Java / Kotlin | `@Async` method; `executor.submit` or `CompletableFuture.runAsync` with no `get` or `join`; `@TransactionalEventListener` + `@Async`; coroutine `launch` in an outer scope | `CompletableFuture.allOf(...).join()`, `invokeAll`, `coroutineScope { async {} }` + `await` | `synchronized`, `ReentrantLock`, `Atomic*`, `@Lock` |
| Python | `asyncio.create_task` with no `await`; threads without `join`; FastAPI or Starlette `BackgroundTasks` (runs after the response); Celery `.delay()` | `await asyncio.gather`, `TaskGroup`, `concurrent.futures` + `result()` | `asyncio.Lock`, `threading.Lock` |
| C# / .NET | `Task.Run` without `await`; `async void`; `_ = DoAsync()` | `await Task.WhenAll` | `lock`, `SemaphoreSlim`, `Interlocked` |
| Reactive (Reactor, RxJava, Rx.js) | nothing runs until `subscribe`; `subscribeOn` or `publishOn` change threads | `zip`, `merge`, `when`, `block` | operators |
| Across processes | broker publish, job enqueue, outbox row, scheduled pickup, outbound webhook | a request and reply over a broker, when the caller blocks for the reply | distributed locks (Redis `SET NX PX`, advisory locks, lock tables) |

## 4. Crossing a process boundary: publish, enqueue, outbox

Publishing is two things:

1. A **synchronous** call to the broker or queue client, inside the caller's sequence. It
   returns or fails, and the caller does something with the result. Record what: returns the
   error, logs and continues, or ignores it.
2. An **asynchronous** delivery to a consumer, later, in another process. Draw an `ASYNC
   BOUNDARY` line, then trace the consumer as a separate sequence starting at its registration
   (the queue or topic subscription, the job handler registration), if it is in the repository.
   If it is not, write `Consumer: UNKNOWN (not in this repository)`.

Never draw consumer steps before the producer's response. Never draw them as if they happen
inside the request. Never use the event name to decide which consumer runs: link producer to
consumer by exchange and routing key and binding, topic, subject, or job name.

Delivery semantics (ack mode, retry, dead-lettering, ordering, idempotency) belong to the
distributed view. Record what the consumer's code does (ack after success, nack with requeue on
error) in its sequence. The cross-process view belongs to `distributed-flow`.

## 5. Spawned background work

For every goroutine, thread, task, or background job started on the path, record:

| Field | Question |
|---|---|
| Spawn point | which step starts it, and is that before or after the response is written? |
| What runs | its call chain (a separate sequence, `B1`...) |
| Context | the request's context (canceled when the request ends, so the work may be cut short) or a detached one (`context.Background()`, a new scope): it outlives the request |
| Join | who waits for it. Usually nobody. Say so. |
| Errors | where its errors go. A `go f()` statement discards `f`'s return values. An un-awaited promise's rejection is unhandled. |
| Panics | Go: a panic in the spawned goroutine crashes the process unless that goroutine recovers. Request-level recovery middleware does not cover it. |
| Lifetime | what happens on shutdown: dropped, drained, or `UNKNOWN` |

Do not claim that background work "completes", "is retried", or "is guaranteed" unless the code
shows it.

## 6. Concurrent branches and joins

Claim concurrency only with a spawn and a join that you read. For each concurrent section,
record:

- **Branches**: what each one calls (arms `4a`, `4b`, with their own steps).
- **Join**: the statement that waits, and its semantics:

| Join | Semantics |
|---|---|
| `errgroup.Wait()` (with `WithContext`) | waits for all branches; returns the first non-nil error; the derived context is canceled on the first error, so branches that honor it stop early |
| `sync.WaitGroup.Wait()` | waits for all; carries no errors; errors must travel another way (channel, shared slice) |
| `Promise.all` | rejects on the first rejection; **the other promises keep running**; their results are dropped |
| `Promise.allSettled` | waits for all; never rejects |
| `asyncio.gather` | first exception propagates by default, and the others keep running; `return_exceptions=True` collects them |
| `CompletableFuture.allOf(...).join()` | waits for all; throws if any failed |

- **Order between branches**: none. Draw them side by side, not one above the other. Do not number
  them as if one happens first.
- **Result use**: how results reach the code after the join (shared variables, channels, return
  values). Say whether a failed branch's partial effect (a write, a call) is undone. Usually it is
  not.

## 7. Shared state and locks

Report synchronization only where it changes the sequence:

- **Critical sections**: which steps run while a mutex or distributed lock is held, where it is
  released on every path (including errors), and what a waiting caller does meanwhile.
- **Distributed locks**: key, TTL, what happens if the lock expires mid-work, and release on every
  exit.
- **Races**: a shared variable written by concurrent branches with no synchronization, or a
  check-then-act across two steps with no lock or guarded update. Describe the concrete
  interleaving, labeled `INFERRED`. Do not report theoretical races the code structure rules
  out. For example, a variable written in one branch and read only after `Wait()` is safe.

## 8. Representation

```text
RefundService.ProcessRefund()
   |
   PAR  errgroup.WithContext(ctx)
   +-- 4a  FraudClient.Score()      HTTP POST /score
   +-- 4b  LimitRepo.ForCustomer()  SELECT refund_limits
   JOIN g.Wait(): all; first error returned; gctx canceled
   |
   v
RefundService.ProcessRefund()
   |  Publish(ctx, "refund.completed", evt)     sync client call; error logged, not returned
   v
[RabbitMQ]
   :
==== ASYNC BOUNDARY: consumer runs later, in cmd/notifier (Sequence A) ====

RefundHandler.Create()
   |  <- 201 Created (response written)
   +~~> go h.notify(context.Background(), r)    Sequence B; not awaited; error discarded
```

In HTML, put concurrent branches in a `frame par` and draw a `boundary` row at every async
boundary. Draw spawned work as an `async` message, and give each separate sequence its own
lifeline diagram under Async / Concurrent Execution.
