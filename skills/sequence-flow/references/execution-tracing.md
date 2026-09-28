# Execution Tracing

How to go from a request ("trace ProcessRefund") to a verified record of what executes: find the
real starting point, walk the code in execution order, capture what matters at each step, and
check the result against the source before writing it up.

## Contents

1. Resolve the request to one starting point
2. Find the entry point by its registration
3. The wrapper chain
4. Walk the code: the trace ledger
5. Reachability
6. Data flow and state mutation
7. Database calls and transactions
8. External calls
9. Validate against source

## 1. Resolve the request to one starting point

| The user gives | Find the start this way |
|---|---|
| Endpoint (`POST /orders`) | the route registration, then the handler it points to (section 2) |
| Function or method (`CreateOrder()`, `RefundService.ProcessRefund`) | its definition. The sequence starts **at that function**. Then find its callers (search the symbol name, interface method name, and DI registrations) and list the entry points that reach it in the Entry Point section. If there are several, name the one the sequence assumes and why. |
| Class | the public method that performs the operation named in the request. If there are several, list them and choose. |
| Feature or use case ("the login flow") | the route, command, or handler whose name, path, or payload matches, confirmed by what it does. When one feature has several requests (login = password check + MFA verify), trace the one that performs the core operation and list the others. |
| Condition or outcome ("what happens when payment succeeds") | the code that reacts to that condition: the webhook `switch` arm for the event type, the consumer of `payment.succeeded`, the branch that writes the `PAID` status. The arm for that condition is the main path; the sequence starts at that arm's entry point. |
| Event or topic (`order.created`) | the consumer subscription (queue binding, topic subscription, handler registration), then the handler. Link by exchange and routing key, topic, or subject, not by a similar name. |
| Job, cron, worker | the scheduler or worker registration, then the job function |
| File | the entry points defined in it. If there is more than one, list them and choose the one that matches the request. |

When the request is thin, search the repository before asking. Ask only when several candidates
remain equally plausible after reading them. If you cannot ask, choose the most likely one and
state the choice and the alternatives in the Overview.

## 2. Find the entry point by its registration

A function with a matching name is not an entry point. Find where the trigger is **registered**,
then open what it points to.

| Trigger | Registration patterns |
|---|---|
| HTTP / REST | Go: `.Post(`, `.POST(`, `HandleFunc(`, `mux.Handle("POST /path"`. Node: `router.post(`, `app.post(`, `@Post(` (NestJS), `fastify.post(`, Next.js `app/**/route.ts` exporting `POST`. JVM: `@PostMapping`, `@RequestMapping(method = POST)`, JAX-RS `@POST`. Python: `@app.post` (FastAPI), `methods=["POST"]` (Flask), `urls.py` + view (Django). Ruby: `config/routes.rb`. PHP: `Route::post(`. .NET: `[HttpPost]`, `app.MapPost(` |
| GraphQL | schema `type Mutation`/`type Query`, resolver maps, gqlgen `*.resolvers.go`, `@Mutation`, `@MutationMapping` |
| gRPC | `.proto` `rpc Name`, then `RegisterXServer(`, `addService(`, `@GrpcService`, then the method implementation |
| CLI | cobra `&cobra.Command{`, urfave/cli, click `@click.command`, argparse subparsers, commander `.command(`, Rake, Django `BaseCommand`, Spring `CommandLineRunner` |
| Cron / scheduled | `cron.AddFunc(`, gocron, `@Scheduled(`, node-cron, Celery beat, sidekiq-cron, Quartz, Kubernetes `CronJob`, serverless `schedule:` |
| Webhook (inbound) | an HTTP route plus signature verification plus a `switch` on event type |
| Message consumer | RabbitMQ `Consume(` + `QueueBind(`, `@RabbitListener`; Kafka `@KafkaListener`, consumer groups, `consumer.run`; NATS `Subscribe(`, `QueueSubscribe(`, JetStream consumers; SQS `ReceiveMessage` |
| Worker / job | Sidekiq `perform`, Celery `@task`, BullMQ `new Worker(`, asynq `HandleFunc`, Hangfire, or an in-process worker started in `main` |

Routes, topics, and job names are often constants or config keys. Search for the literal string
**and** the constant name. The same trigger can be registered in more than one process; the
deployable that registers it is part of the entry point.

Record:

```text
ENTRY
  Trigger       POST /refunds
  Registered    internal/http/routes.go Routes()            r.Post("/refunds", h.Create)
  File          internal/http/refund.go
  Function      RefundHandler.Create
  Process       cmd/api (main.go builds the router)
```

## 3. The wrapper chain

Code runs before the handler and after it. Middleware, interceptors, guards, filters,
decorators, and aspects wrap the handler like layers. Execution enters the outermost layer
first and leaves it last.

```text
Tracing -> RateLimit -> RequireAuth -> [ RefundHandler.Create ] -> RequireAuth -> RateLimit -> Tracing
  in         in           in                                          out           out         out
```

- Record the chain in execution order, and only the layers that apply to **this** route. A route
  group applies its middleware only to routes registered inside it.
- Mark the layers that can end the request before the handler runs (auth `401`, rate limit
  `429`, validation `400`), and the layers that act on the way out (panic recovery, error
  mapping, a per-request transaction commit, response logging).
- Framework plumbing that does not change behavior (routing, header parsing) needs no step.

## 4. Walk the code: the trace ledger

Walk depth-first, in execution order. Read each function on the path **in full**, not just the
lines around the call you followed in. For each statement that matters, add one ledger line:

```text
<id>  <file> <Symbol>  <KIND>  <what happens>                         <LABEL>
```

`KIND` is one of `CALL`, `RETURN`, `DECISION`, `LOOP`, `PAR`, `JOIN`, `DB`, `EXT`, `SPAWN`,
`PUBLISH`, `STATE`, `ERROR`, `DEFER`. The ledger is your working notes. It becomes the step table
and the sequence model, so you never need to re-derive the order from memory.

For each call you meet, decide:

| The call | Do |
|---|---|
| Changes the outcome (decides, writes, calls out, transforms the result, can fail in a way that is handled) | resolve the callee (`call-chain.md`), then expand it |
| Leaves the process (database, cache, broker, external API) | record the operation (sections 7 and 8), do not go inside the client library |
| Hands work to something that is not awaited | record the handoff, draw the boundary (`async-boundary.md`), and trace the other side as a separate sequence |
| Trivial helper, logging, metrics | skip it, or give it one line if it runs I/O |

At every function, also read its error handling (`control-flow.md`), its decisions
(`branching.md`), and its loops (`loops.md`) before moving on.

## 5. Reachability

Include code in the sequence only if it executes on this path. Check that it is:

- **called or registered** from the path you are on, not just present in the repository
- **the implementation wired into this process**, not a stub, fake, or alternative
- **not disabled** by a feature flag, build tag, profile, or environment check (in-repo default:
  `CONFIRMED`; deployed value: `UNKNOWN`)
- **not test-only**, and not a `TODO` or placeholder that returns immediately

Code that looks relevant but does not execute (an unused validator, an unwired implementation, an
unreachable `case`, a state constant this flow never writes) goes in **Not on this path**, with
the reason. When documentation says such code is used, it is also **documentation drift**.

## 6. Data flow and state mutation

**Data flow.** Track only the data that steers execution or forms the result: the values used in
decisions, the values written, and the values returned. Record each shape it takes and the
function that transforms it:

```text
HTTP body -> RefundRequest -> refund.Command -> refund.Refund -> SQL parameters -> refunds row
refunds row -> refund.Refund -> RefundResponse -> JSON
```

- Record a transformation only where the code performs one (a mapper, a constructor, a `Scan`, a
  serializer). Do not add layers the code does not have.
- Note where a value that drives a later decision is **computed**. A total calculated at step 5
  and charged at step 7 links the two steps.
- Note lossy or surprising mappings: a field that is dropped, a default filled in, an
  unrecognized value silently ignored.

**State mutation.** For each business state field (status, balance, flags):

1. Find the states: enums, constants, check constraints in migrations.
2. Find every write of each state on this path: assignments, setter methods, `UPDATE ... SET
   status`.
3. For each change, record from, to, the step and function responsible, and **when it becomes
   durable**. An in-memory change (`r.Complete()`) is not persisted until a later write. If
   something fails between the two, the change is lost. Record both steps.
4. A state that is declared but never written by this operation is "not reached by this
   operation". Do not guess which flow sets it.

## 7. Database calls and transactions

At the sequence level, each database call records:

```text
<id>  <Symbol> -> [<store>]  <OPERATION> <table>   tx: <T1 | autocommit>   check: <what the code checks>
```

- **Operation** is one of `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `UPSERT`, `LOCK`, or a
  transaction statement (`BEGIN`, `COMMIT`, `ROLLBACK`).
- **Statement source.** For literal SQL, quote the verb and table, and the `WHERE` or `SET` part
  when it drives the outcome. For an ORM call, write the ORM call (`db.Create(&r)`,
  `repo.save(r)`) and the operation it implies. **Never invent SQL text the code does not
  contain.**
- **Check**: what the code does with the result: rows affected compared to zero, no-rows mapped
  to not-found, the error ignored, a unique-violation code handled. An unchecked result on a
  guarded `UPDATE` (`WHERE status = 'PENDING'`) means a lost race goes unnoticed. That is a
  finding.

**Transactions go inside the sequence.** Draw `BEGIN` where the transaction starts, the statements
in their order, `COMMIT` where it ends, and each exit:

- every error return between `BEGIN` and `COMMIT`, and which rollback runs (deferred, explicit,
  framework)
- a return path that neither commits nor rolls back (a leaked transaction)
- non-database side effects **inside** the transaction (HTTP calls, publishes, emails), which a
  rollback cannot undo
- what runs **outside** the transaction, before or after it. Separate autocommit writes can leave
  partial state.

Also look for transactions you cannot see at the call site: per-request transaction middleware,
`@Transactional` (and its self-invocation trap), Django `ATOMIC_REQUESTS`, unit-of-work wrappers.

## 8. External calls

For each call that leaves the process to another system:

```text
<id>  <client Symbol> -> [<target>]  <PROTOCOL> <method + path | rpc | command>
      request:   the fields that matter
      response:  what the caller uses
      errors:    status or error -> what the client returns (ErrX), and what the caller then does
      timeout:   as configured in code, or none found           retry: as in code, or none found
```

- **Target**: name it by what the code uses, the config key or host (`PSP (PSP_URL)`). Do not
  guess a vendor.
- **Protocol**: `HTTP`, `REST`, `GRPC`, `GRAPHQL`, `WEBHOOK`, or the SDK (`AWS SQS SDK`).
- Record timeout and retry only as the code shows them: the client's construction, a context
  deadline, a retry wrapper. A comment, README, or name that says "retries" is not evidence. Retries added by
  infrastructure the repository cannot show are `UNKNOWN`.
- The receiver's internals are `UNKNOWN` unless its code is in this repository. If it is, link to
  its entry point; do not expand it unless the user asked for the cross-service sequence.

## 9. Validate against source

Before writing the artifacts, walk the ledger once more with the code open. This is a separate
pass, not a memory check.

1. **Entry point**: the registration line exists and points at this handler, in this process.
2. **Each call**: the call site exists in the caller, and the callee is the implementation wired
   on this path.
3. **Order**: each step comes after the previous one in the code, or after a join or return that
   links them. Nothing from a spawned goroutine or a consumer is placed inside the synchronous
   sequence.
4. **Branches**: every decision on the path is present, with all arms that have distinct outcomes,
   and each arm's reachability is stated.
5. **Error paths**: each error's propagation matches the code at every layer, and the final
   mapping matches the error-to-response table, including the default.
6. **Return path**: every transformation is a function you read.
7. **Async and concurrency**: every `PAR` has a spawn and a join you read. Every boundary has the
   spawn or handoff you read, and nothing after it is drawn as synchronous.
8. **Database and transactions**: every statement and its transaction membership match the code.
   No SQL was invented.
9. **External calls**: the client, protocol, target, and error mapping match the code. Timeouts
   and retries are recorded only as the code shows them.
10. **Evidence**: every `CONFIRMED` step cites a file and symbol you read, and line numbers only
    where you read those lines.
11. **Labels**: anything you could not point at in code is `INFERRED` (with reasoning) or
    `UNKNOWN` (with how to resolve it), or it is removed.

Fix the ledger, then write the artifacts from it.
