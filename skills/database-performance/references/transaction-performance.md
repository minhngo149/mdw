# Transaction Performance

What a database transaction costs while it is open, how to map what runs inside it, and which
patterns make transactions long, large, or wasteful. This reference is about **cost**: duration,
connections, locks, and commit overhead. What a transaction makes atomic, dual writes, outboxes,
and sagas are correctness topics, not cost. Do not re-derive them here.

## Contents

1. What an open transaction costs
2. Mapping a transaction
3. Work that should not run inside a transaction
4. Implicit and per-request transactions
5. Transactions that are too small
6. Isolation, retries, and contention
7. Error paths that leave transactions open
8. Validation

## 1. What an open transaction costs

From `BEGIN` to `COMMIT`/`ROLLBACK`, a transaction:

- **holds one pool connection** for its whole duration, whether or not a statement is running
- **holds every row lock it has taken** (from `UPDATE`, `DELETE`, `INSERT` into unique indexes,
  `SELECT ... FOR UPDATE/SHARE`) until it ends; other writers of those rows wait
  (`locking-concurrency.md`)
- in **PostgreSQL**, holds back the cleanup horizon: `VACUUM` cannot remove row versions that
  the open transaction might still need, so long transactions cause table and index bloat on
  busy tables. A session that stops sending statements without ending its transaction shows as
  `idle in transaction`
- in **MySQL InnoDB**, keeps undo history alive (the history list length grows and purge lags);
  under the default `REPEATABLE READ`, the read view stays open
- delays replication and change-data-capture consumers that wait for the commit

Its duration is the sum of every statement **plus all application work between the statements**:
network calls, CPU, sleeps, retries, and waiting for other locks. The application work is
usually the larger part and the part static analysis can see.

## 2. Mapping a transaction

For each transaction on an important path, write:

```text
TX-2  internal/billing/service.go SettlementService.Settle        connection held throughout
BEGIN                                                             (db.BeginTx with request ctx)
  SELECT ... FROM accounts WHERE id = $1 FOR UPDATE               locks 1 accounts row
  for each open invoice (N, unbounded):
    UPDATE invoices SET status = 'settled' WHERE id = $1          locks N invoices rows
  POST {LEDGER_URL}/entries                                       remote call, client timeout 30 s
  INSERT INTO settlements (...)
COMMIT
Statements  2 + N                         CONFIRMED
Locks held  1 account + N invoices until COMMIT                   CONFIRMED
Duration    statements + ledger call; bounded only by the 30 s client timeout and the request
            context deadline (UNKNOWN)                            INFERRED
```

Find the boundary:

| Stack | Where it starts and ends |
|---|---|
| Go | `db.BeginTx`/`pool.Begin` to `tx.Commit`/`tx.Rollback`; GORM `db.Transaction(func(tx) error)`; Ent `client.Tx` |
| Node | Prisma `$transaction(async (tx) => ...)` (interactive) or `$transaction([...])` (batch); Knex `knex.transaction`; TypeORM `manager.transaction`/`queryRunner.startTransaction`; Sequelize `sequelize.transaction`; raw `BEGIN` on a checked-out client |
| JVM | `@Transactional` method boundaries (proxy-based: self-invocation bypasses it), `TransactionTemplate` |
| Python | Django `transaction.atomic`, SQLAlchemy `session.begin()` / the implicit transaction until `commit()`, asyncpg `conn.transaction()` |
| Ruby, PHP, .NET | `ActiveRecord::Base.transaction`, `DB::transaction`, `BeginTransaction`/`TransactionScope` |

Everything the code does between the start and the end is inside, **including calls that do
not use the transaction handle**. In Go, a function that makes an HTTP call while the caller
holds `tx` runs inside the transaction even though it never touches `tx`. In `@Transactional`
methods, every callee runs inside.

## 3. Work that should not run inside a transaction

Report each of these only when you traced it between the start and the end on a real path:

| Work | Why it lengthens the transaction |
|---|---|
| HTTP, gRPC, or other remote calls | duration bounded only by the client timeout; a rollback cannot undo the remote effect |
| Broker publishes that wait for a confirmation | network round trip plus broker latency |
| Cache or search-index round trips | usually short, but each is a network hop |
| File or object-storage I/O, email, SMS | slow and not rolled back |
| Retry loops with backoff or sleeps | multiply the duration |
| Waiting for a mutex, channel, semaphore, or another lock | unbounded by the database |
| Heavy CPU: PDF rendering, image processing, password hashing | holds locks while computing |
| Large result processing (iterate and transform thousands of rows) | duration grows with data |

Frame the finding with its bound. Holding row locks across a remote call means the locks and
the connection are held for as long as the remote call takes, up to its timeout. Every concurrent
request needing the same rows waits that long. Every concurrent request of this kind occupies a
connection that long.

**Improvement options change correctness.** Moving the remote call before `BEGIN` (validate
first, then write) or after `COMMIT` (with a pending state, an outbox, or reconciliation) changes
what is atomic. Say so explicitly and point to the correctness analysis. Never recommend moving a
call out of a transaction as a pure performance fix.

## 4. Implicit and per-request transactions

Transactions that no call site shows:

| Mechanism | Effect |
|---|---|
| Spring Boot `spring.jpa.open-in-view` (default `true`; Spring logs a warning at startup) | the persistence context stays open for the whole web request, so lazy loads run during serialization, and the connection can stay held until the response is written (`INFERRED`; depends on connection-release settings) |
| Django `ATOMIC_REQUESTS = True` | every view runs in one transaction, including its slow parts |
| Unit-of-work middleware (custom) | same as above; read the middleware |
| Class-level `@Transactional` | every public method opens a transaction, including reads; `readOnly = true` lets Hibernate skip dirty checking |
| GORM default transactions | every `Create`/`Update`/`Delete` runs in its own transaction unless `SkipDefaultTransaction: true`; extra round trips per write |
| Hibernate auto-flush | before a query, pending changes are flushed; large persistence contexts make every query pay for dirty checking |
| SQLAlchemy implicit transaction | a `Session` begins a transaction on first use and holds it until `commit()`/`rollback()`/`close()` |
| Rails `after_save` vs `after_commit` | `after_save` callbacks run inside the transaction; slow work there lengthens it |
| EF Core `SaveChanges` | one transaction per call, batching the changes |

## 5. Transactions that are too small

N writes in a loop with no transaction are N commits. Each commit waits for a durable log
flush under default durability settings (PostgreSQL `synchronous_commit = on`, InnoDB
`innodb_flush_log_at_trx_commit = 1`), so the loop pays N flushes (`INFERRED` from the settings;
deployed values are `UNKNOWN`). Grouping the writes into one transaction or into chunks removes
most of that cost.

The trade-off runs the other way from section 3: a larger transaction holds more locks for
longer and grows the undo or bloat footprint. For backfills and bulk jobs, chunk: commit every K
rows, where K is a validated number, not a guess.

Also check that the loop's writes do not need to be atomic together. If they do, one transaction
is a correctness requirement.

## 6. Isolation, retries, and contention

| Engine | Default isolation | Notes |
|---|---|---|
| PostgreSQL | `READ COMMITTED` | `REPEATABLE READ` and `SERIALIZABLE` raise serialization failures (SQLSTATE `40001`) that the application must retry |
| MySQL InnoDB | `REPEATABLE READ` | locking reads and writes take next-key and gap locks (`locking-concurrency.md`) |
| SQL Server | `READ COMMITTED` (locking) | readers block writers unless `READ_COMMITTED_SNAPSHOT` is on (database setting, usually `UNKNOWN`) |
| SQLite | serializable, one writer at a time | `SQLITE_BUSY` under concurrent writes; WAL mode lets readers proceed |

Retries: check that the code retries serialization failures and deadlocks where it uses
isolation or locking that produces them, and how. A retry re-runs the whole transaction, so
duration and load multiply by the attempt count; under contention, retries add more contention.
Record the attempts and the backoff.

A MySQL trap: on a lock wait timeout (error `1205`, default `innodb_lock_wait_timeout` 50 s),
InnoDB by default rolls back only the failed **statement**, not the transaction
(`innodb_rollback_on_timeout = OFF`). Code that catches the error and continues may commit a
partial transaction.

## 7. Error paths that leave transactions open

A transaction left open is a leaked connection plus locks held indefinitely.

| Stack | Check |
|---|---|
| Go `database/sql` | every return path after `BeginTx` reaches `Commit` or `Rollback`; the idiom `defer tx.Rollback()` is safe after a commit. If the context passed to `BeginTx` is canceled, `database/sql` rolls the transaction back, so a transaction begun with the request context is bounded by the request. One begun with `context.Background()` is not |
| pgx | `defer tx.Rollback(ctx)`; the same context rule applies |
| Node `pg` | manual `BEGIN` on `pool.connect()` needs `ROLLBACK` on error **and** `client.release()` in `finally` |
| Prisma, Knex, Sequelize managed callbacks | commit and rollback are handled by the library (`INFERRED`); check for work that escapes the callback |
| Python | `with transaction.atomic():`, `with session.begin():`; manual `begin()` without `finally` |
| JVM | declarative transactions roll back only on unchecked exceptions unless `rollbackFor` says otherwise, so a caught checked exception commits (correctness) |

**Nested transactions and savepoints.** Each savepoint costs a round trip. In PostgreSQL, a
backend with more than 64 open subtransactions (savepoints, or PL/pgSQL `EXCEPTION` blocks)
overflows its subtransaction cache, which is a known cause of severe slowdowns on busy systems.
Report it only when a loop creates savepoints (for example nested `atomic()` or
`requires_new: true` inside a loop).

## 8. Validation

- PostgreSQL: `pg_stat_activity` (`state = 'idle in transaction'`, `now() - xact_start`),
  `idle_in_transaction_session_timeout` as a guard, `pg_stat_user_tables.n_dead_tup` for bloat.
- MySQL: `information_schema.innodb_trx` (`trx_started`), `SHOW ENGINE INNODB STATUS` (history
  list length).
- Application: tracing spans around each transaction, with the remote calls inside visible as
  child spans; transaction duration percentiles.
- Load test the path with concurrent requests for the same rows, and record lock waits
  (`locking-concurrency.md` section 10) and pool wait time (`connection-pool.md` section 8).
