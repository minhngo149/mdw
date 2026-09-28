# Transaction Flow

Transactions decide which state changes succeed or fail together. Getting their boundaries
wrong is the most common way flow documentation misleads readers. Report only what the code
shows. Most flows have fewer guarantees than their documentation suggests.

## Contents

1. Database transaction vs. distributed business transaction
2. Finding database transactions
3. Mapping the boundary
4. Locks and concurrency control
5. Distributed patterns and the evidence each requires
6. Dual writes
7. State transitions

## 1. Database transaction vs. distributed business transaction

| | Database transaction | Distributed business transaction |
|---|---|---|
| Scope | one connection to one database | several processes, stores, or external systems |
| Guarantee | atomicity and isolation provided by the database | none by default. Consistency is designed in (saga, outbox, compensation, reconciliation) or absent. |
| Evidence | `BEGIN`/`COMMIT`/`ROLLBACK`, a transaction object, `@Transactional` | orchestration code, compensating actions, an outbox relay, persisted saga state |

Never call a multi-step flow "transactional" without saying which kind of transaction you
mean. Most business flows are a sequence of separate commits and remote calls.

## 2. Finding database transactions

| Stack | Explicit transaction APIs |
|---|---|
| Go | `db.BeginTx`, `pool.Begin`, `tx.Commit`, `tx.Rollback`, GORM `db.Transaction(func(tx) ...)`, ent `client.Tx` |
| Node | `knex.transaction`, `prisma.$transaction`, `sequelize.transaction`, TypeORM `manager.transaction` / `queryRunner.startTransaction`, `client.query('BEGIN')`, mongoose `session.withTransaction` |
| JVM | `@Transactional`, `TransactionTemplate`, `EntityManager.getTransaction()` |
| Python | Django `transaction.atomic`, SQLAlchemy `session.begin()`, asyncpg `conn.transaction()` |
| Ruby | `ActiveRecord::Base.transaction`, `Model.transaction` |
| .NET | `BeginTransaction`, `TransactionScope`, EF `SaveChanges` (one transaction per call) |

Also check for transactions you cannot see at the call site:

- **Per-request transactions:** unit-of-work middleware, Django `ATOMIC_REQUESTS`.
- **Declarative pitfalls (Spring and similar proxy-based frameworks):** a self-invoked
  `@Transactional` method bypasses the proxy, so no transaction starts. Rollback happens only on
  unchecked exceptions unless `rollbackFor` says otherwise. Propagation (`REQUIRES_NEW`,
  `NESTED`) and `readOnly` change behavior. Annotations on private methods are ignored.
- **No transaction at all:** each statement autocommits. Several writes with no transaction
  can leave partial state. You read the writes, so their independence is `CONFIRMED`; the
  partial-state consequence is `INFERRED`.

## 3. Mapping the boundary

For each database transaction, record where it starts and ends, the operations inside it, the
isolation level and locks (if set), what triggers a rollback, and what runs **after** the
commit. Then list every side effect **inside** the transaction that the rollback cannot undo:
HTTP calls, broker publishes, cache writes, emails, file writes.

```text
T1  internal/booking/repo.go Reserve()
BEGIN
  SELECT seats WHERE id = ANY($1) FOR UPDATE          LOCK
  UPDATE seats SET held_by = $2                       UPDATE
  INSERT INTO bookings (status = 'HELD')              INSERT
COMMIT
-- no transaction below this line --
POST {PSP_URL}/payments                               remote side effect
UPDATE bookings SET status = 'CONFIRMED'              autocommit
```

Rollback checks:

- Is there an explicit rollback on every error path, or a deferred rollback (Go
  `defer tx.Rollback()` is a no-op after a successful commit)?
- Is there a path that returns without committing or rolling back (a leaked transaction)?
- An error **after** commit, such as a failed response write, does not roll anything back.

Nested transactions: PostgreSQL `SAVEPOINT`, nested Django `atomic` blocks, Spring `NESTED`,
and nested GORM `Transaction` calls use savepoints. Record the propagation you observe.

## 4. Locks and concurrency control

| Mechanism | Evidence to find | Check |
|---|---|---|
| Pessimistic lock | `SELECT ... FOR UPDATE / FOR SHARE / SKIP LOCKED`, `LOCK TABLE`, JPA `@Lock(PESSIMISTIC_WRITE)`, `pg_advisory_xact_lock` | Is it inside the transaction whose writes it protects? |
| Optimistic lock | a version column plus `WHERE version = ?`, JPA `@Version`, ORM optimistic-lock options | Is the affected row count checked? Without that check, conflicts are not detected. |
| Unique constraint as concurrency control | a `UNIQUE` index in migrations plus code handling the violation (`23505`, `DuplicateKey`, `IntegrityError`) | What does the handler do: return the existing row, return 409, retry? |
| Guarded state transition | `UPDATE ... WHERE status = 'PENDING'` plus a check of the affected row count | Is the zero-rows case handled? |
| Distributed lock | Redis `SET NX PX`, Redlock, etcd, ZooKeeper, a lock table | TTL, release on every path, and behavior when the lock expires mid-work |
| Idempotency record | a table or key storing request or message IDs | Scope, TTL, when it is written relative to the work, and whether it is released on failure |

Report a mechanism only with evidence. A missing mechanism is a finding only when a concurrent
interleaving can actually break the flow; describe that interleaving.

## 5. Distributed patterns and the evidence each requires

Name a pattern only when **all** of its required evidence is present on this flow's path.

| Pattern | Required evidence | If only part is present |
|---|---|---|
| Transactional outbox | (1) an insert into an outbox table in the **same** database transaction as the business write, **and** (2) a relay (poller, CDC or Debezium config) that publishes the rows and marks them sent | Write found but no relay: "outbox write CONFIRMED, relay UNKNOWN". Only a type, table, or TODO: not an outbox on this path. If docs claim one, record drift. |
| Inbox | the consumer records the message ID in the same transaction as its side effect, and checks it before acting | a dedupe check outside the transaction is best-effort; say so |
| Orchestrated saga | a coordinator that issues steps, persists saga state, and invokes compensations on failure | steps without compensations are a sequence, not a saga |
| Choreographed saga | an event chain in which failure events trigger compensating handlers in other participants | an event chain without compensation is not a saga |
| Compensation | code that semantically undoes a completed step (refund, release, cancel) after a later failure | a status update such as `FAILED` is not compensation unless it reverses an effect |
| 2PC / XA | an XA datasource, a JTA transaction manager, Atomikos or Narayana, Seata AT/TCC | rare; never infer it |
| Reconciliation | a scheduled job that finds and repairs inconsistent state | none found: say "no recovery mechanism found in repository" |

## 6. Dual writes

When a flow writes to two systems without a coordinating pattern, find the order of the writes
and the gap between them:

| Order | If the second write fails or the process dies in between |
|---|---|
| database commit, then publish | the state is saved but the event is missing |
| publish, then database commit | consumers act on state that may be rolled back |
| external call, then database write | a remote effect exists with no local record |
| database write, then external call | a local record exists with no remote effect |

You read the order, so the order is `CONFIRMED`; the consequence is `INFERRED`. Then look for
anything that detects or repairs the gap: a retry, reconciliation, or a status that a sweeper
job picks up. If there is none, say so.

## 7. State transitions

- Find the states in enums, constants, and database check constraints.
- Find the transitions by searching for each state constant among the writes.
- For each transition record: from, to, triggered by (function or event), guard (a `WHERE`
  status condition, if any), the transaction it belongs to, and its evidence label.
- A state that is declared but never written by this flow: say it is not reached by this flow.
  Do not guess which flow sets it.
- A state the flow can get stuck in (written before a remote call, never advanced on some
  failure path) is a key finding. Link it to the failure path that causes it.

```text
| From | To        | Trigger                        | Guard                | Tx | Evidence                             | Label     |
|------|-----------|--------------------------------|----------------------|----|--------------------------------------|-----------|
| -    | HELD      | BookingService.Reserve()       | -                    | T1 | internal/booking/repo.go Reserve()   | CONFIRMED |
| HELD | CONFIRMED | BookingService.Confirm()       | WHERE status='HELD'  | -  | internal/booking/repo.go Confirm()   | CONFIRMED |
| HELD | EXPIRED   | ExpireHolds cron               | held_at < now()-15m  | -  | internal/jobs/expire.go Run()        | CONFIRMED |
```
