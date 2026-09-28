# Locking and Concurrency

How to find the locks the code takes (explicit, implicit, application-level, distributed),
how long each is held, where contention and deadlock risk come from, and how DDL in migrations
locks tables. A lock is a correctness tool first. Never recommend removing one for performance
without naming what replaces its guarantee.

## Contents

1. Lock inventory
2. Row locks, explicit and implicit
3. Hot rows and contention
4. Lock ordering and deadlock risk
5. Table locks and migrations
6. Advisory, application, and distributed locks
7. Engine differences
8. Optimistic concurrency and retries
9. Queue tables
10. Validation

## 1. Lock inventory

Record each lock on an important path:

```text
LOCK  accounts row (by id)          FOR UPDATE      acquired SettlementService.Settle   released COMMIT (TX-2)
      held across: N invoice UPDATEs, POST {LEDGER_URL}/entries (30 s timeout)                    CONFIRMED
LOCK  invoices rows (N, order of the open-invoices query)   UPDATE   same TX, same release        CONFIRMED
```

A lock's duration runs from the statement that takes it to the end of the transaction, not
to the end of the statement. The transaction map (`transaction-performance.md` section 2) gives
you what the lock is held across.

## 2. Row locks, explicit and implicit

**Explicit.** `SELECT ... FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE`, `FOR KEY SHARE`
(PostgreSQL); `FOR UPDATE`, `FOR SHARE` / `LOCK IN SHARE MODE` (MySQL); modifiers `NOWAIT` and
`SKIP LOCKED` (PostgreSQL 9.5+, MySQL 8.0+). ORM forms: GORM
`Clauses(clause.Locking{Strength: "UPDATE"})`, JPA `@Lock(PESSIMISTIC_WRITE)`, Django
`select_for_update()`, Rails `lock`/`lock!`/`with_lock`, Sequelize `lock: true`, Laravel
`lockForUpdate()`/`sharedLock()`. Prisma and EF Core have no row-lock API, so their locks come
from raw SQL.

**Implicit.**

- Every `UPDATE` and `DELETE` locks the rows it changes until the transaction ends.
- An `INSERT` of a key that another uncommitted transaction has just inserted into a unique
  index waits for that transaction.
- Foreign-key checks lock the parent row: PostgreSQL takes `FOR KEY SHARE` on the referenced row
  when a child row is inserted or its key updated (this does not block non-key updates of the
  parent); InnoDB takes a shared lock on the parent record.
- **InnoDB scans lock what they read.** Under `REPEATABLE READ` (the default), locking reads and
  `UPDATE`/`DELETE` lock every index record they scan, plus the gaps between them (next-key
  locks). An `UPDATE ... WHERE` whose predicate no index serves scans and locks the whole table
  for the duration of the transaction. In MySQL, a missing index is a locking problem as well as
  a read problem. PostgreSQL locks only the rows actually modified.

## 3. Hot rows and contention

Contention happens when many concurrent transactions need the same row. Look for rows that
every request of some kind writes:

| Hot-row pattern | Example |
|---|---|
| Counters and aggregates | `UPDATE stats SET views = views + 1`, Rails `counter_cache` |
| Balances, quotas, inventory per item | `UPDATE wallets SET balance = balance - $1 WHERE id = $2` |
| Sequence or "next number" rows | invoice numbering tables |
| Parent timestamps touched on every child write | Rails `touch: true`, `UPDATE projects SET updated_at = now()` |
| Singleton rows | global settings, a rate-limit row, a "last run" row |
| Popular entities | one tenant, item, or event that most traffic targets |

Waiting time for a hot row equals the holder's transaction duration, so a hot row inside a long
transaction is the combination to report. Static analysis can confirm that the row is written on
every request of a kind and how long the lock is held; how many requests collide is `UNKNOWN`.

Options, each with trade-offs: shorten the transaction; a single atomic `UPDATE ... SET x = x +
1` instead of read-modify-write; sharded or bucketed counters; aggregating increments
asynchronously (eventual consistency); `SKIP LOCKED` for work distribution.

## 4. Lock ordering and deadlock risk

A deadlock needs two transactions that each hold a lock the other wants. Report **potential
deadlock risk** when code shows one of these, and describe the interleaving:

| Evidence | Interleaving |
|---|---|
| A loop locks rows in an order taken from input or from an unordered query (request items, a map, a set) | T1 locks A then wants B; T2 locks B then wants A |
| Two code paths lock the same tables in opposite orders | path P: accounts then invoices; path Q: invoices then accounts |
| A multi-row `UPDATE`/`DELETE` or `FOR UPDATE` without `ORDER BY` over overlapping row sets | two statements lock the shared rows in different physical orders |
| InnoDB gap locks plus concurrent inserts into the same range | insert-intention waits on each other's gap locks |
| Parent and child rows locked in different orders through foreign-key checks | child insert locks the parent; another transaction locks the parent, then the child |

Engines detect deadlocks and abort one transaction (PostgreSQL after `deadlock_timeout`, 1 s by
default; InnoDB immediately with `innodb_deadlock_detect` on). The victim gets an error:
PostgreSQL `40P01`, MySQL `1213`, SQL Server `1205`. Check whether the code retries it. The usual
mitigation is a consistent lock order (sort IDs; `ORDER BY id FOR UPDATE`) and shorter
transactions.

Never state that a deadlock occurs without runtime evidence (logs, `LATEST DETECTED DEADLOCK`).

## 5. Table locks and migrations

`LOCK TABLE` in application code serializes all access; report it with its transaction scope.

DDL in migrations runs against live tables at deploy time:

| PostgreSQL operation | Lock behavior |
|---|---|
| Most `ALTER TABLE` forms | `ACCESS EXCLUSIVE`: blocks reads and writes. While it waits behind a long transaction, it also blocks every query queued behind it. Set `lock_timeout` in migrations |
| `CREATE INDEX` (not `CONCURRENTLY`) | blocks writes for the whole build |
| `ADD COLUMN` with a volatile default, or any default before PostgreSQL 11; `ALTER COLUMN TYPE` (most changes) | rewrites the table under `ACCESS EXCLUSIVE` |
| `ADD CONSTRAINT ... FOREIGN KEY` / `CHECK` | scans to validate; use `NOT VALID`, then `VALIDATE CONSTRAINT` |
| `SET NOT NULL` | scans the table (PostgreSQL 12+ can skip the scan when a valid `CHECK (col IS NOT NULL)` exists) |

MySQL: online DDL support varies by operation (`ALGORITHM=INSTANT` adds columns from 8.0.12,
at any position from 8.0.29), and every DDL needs a metadata lock that queues behind open
transactions and blocks queries behind it. gh-ost and pt-online-schema-change exist for large
tables. SQLite rewrites tables for most alterations.

The repository cannot tell which migrations have already run in production. Report migration
locking as a pattern for migrations touching tables with growth signals, with table size
`UNKNOWN`.

## 6. Advisory, application, and distributed locks

| Mechanism | Check |
|---|---|
| PostgreSQL `pg_advisory_lock` (session) | held until `pg_advisory_unlock` or the session ends. Returned to the pool still locked if the unlock is missed. Broken under PgBouncer transaction pooling |
| PostgreSQL `pg_advisory_xact_lock` | released at transaction end; the safer form |
| MySQL `GET_LOCK(name, timeout)` | session-level; same pool caveat |
| In-process mutex (`sync.Mutex`, `synchronized`, `threading.Lock`) around database work | protects one process only. With several instances it protects nothing across them. Holding it during a query serializes every request through that section, so throughput through it is at most one call per query latency |
| Distributed lock (Redis `SET NX PX`, Redlock, etcd, ZooKeeper, a lock table) | TTL against the work's duration, release on every path, behavior when the TTL expires mid-work. Correctness details belong to the `distributed-flow` analysis |
| Leader election for scheduled jobs | without it, a job started in every replica runs once per replica |

## 7. Engine differences

| | PostgreSQL | MySQL InnoDB | SQLite | SQL Server |
|---|---|---|---|---|
| Default isolation | `READ COMMITTED` | `REPEATABLE READ` | serializable, one writer | `READ COMMITTED` (locking) |
| Plain reads block writers | no (MVCC) | no (consistent reads) | writer blocks readers in rollback-journal mode; not in WAL mode | yes, unless read-committed snapshot is on |
| Gap or range locks | no (predicate locks only at `SERIALIZABLE`) | yes, at `REPEATABLE READ` | n/a | at `SERIALIZABLE` |
| Lock wait limit | `lock_timeout` (0 = wait forever by default) | `innodb_lock_wait_timeout` (50 s); only the statement rolls back by default | `busy_timeout` | `LOCK_TIMEOUT` (-1 = wait forever) |
| Error codes | `40P01` deadlock, `55P03` lock not available, `40001` serialization failure | `1213` deadlock, `1205` lock wait timeout | `SQLITE_BUSY` | `1205` deadlock victim, `1222` lock timeout |

## 8. Optimistic concurrency and retries

Optimistic locking (a version column with `WHERE version = ?`, JPA `@Version`, Rails
`lock_version`) takes no lock, but it must check the affected-row count, or conflicts go
undetected. Under contention on a hot entity, retries re-run the reads and the writes, so load
grows with the conflict rate. Record the retry count, the backoff, and which errors trigger it.

## 9. Queue tables

For database-backed job queues (`SELECT ... WHERE status = 'pending' ORDER BY id LIMIT n`):

- without `FOR UPDATE SKIP LOCKED`, workers either block each other or claim the same job
- polling frequency times worker count gives the query rate (both from config, or `UNKNOWN`)
- an index for the claim query; in PostgreSQL, a partial index on the pending state fits well
- status updates on a busy table generate dead rows in PostgreSQL (bloat); finished jobs need a
  cleanup or retention job

## 10. Validation

- PostgreSQL: `pg_locks` joined with `pg_stat_activity` (`wait_event_type = 'Lock'`),
  `pg_blocking_pids(pid)`, `log_lock_waits = on` (logs waits longer than `deadlock_timeout`),
  deadlock reports in the server log.
- MySQL: `performance_schema.data_locks` and `data_lock_waits` (8.0), `sys.innodb_lock_waits`,
  `SHOW ENGINE INNODB STATUS` (`LATEST DETECTED DEADLOCK`), `innodb_print_all_deadlocks`.
- SQL Server: `sys.dm_tran_locks`, deadlock graphs from the `system_health` session.
- Reproduce a suspected ordering deadlock with two sessions executing the statements in the
  interleaved order. Load test hot-row paths with concurrent requests for the same key, and
  record lock wait time.
