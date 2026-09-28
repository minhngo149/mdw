# Connection Pool, Leaks, and Timeouts

How to read the pool configuration, compare it with what the repository says about concurrency
and deployment, find code paths that hold or leak connections, and check whether database work
can be canceled. Pool sizing is where invented numbers are most tempting. The repository
usually supports arithmetic on configured values and nothing more.

## Contents

1. What to find
2. Pool defaults by stack
3. Sizing arithmetic without inventing workload
4. What holds a connection
5. Connection and cursor leaks
6. Context, cancellation, and timeouts
7. Poolers and proxies
8. Validation

## 1. What to find

For every deployable that talks to the database:

```text
POOL  <deployable>  <library>  constructed at <file Symbol>
  max            <value | default N (INFERRED, library vX)>  source <code | config key | env default>
  min / idle     ...
  lifetime       ...
  idle timeout   ...
  acquire wait   <timeout | none>
  instances      <replicas from manifests | UNKNOWN>
  processes      <workers per instance: gunicorn workers, Puma workers, PM2/cluster | 1 | UNKNOWN>
  concurrency    <HTTP server threads, worker concurrency, goroutines per job | UNKNOWN>
```

Values set from environment variables: cite the code that reads them and the in-repo default,
and note that deployment can override them.

## 2. Pool defaults by stack

Use these only when the code does not set a value, and label the result `INFERRED` with the
library and version. Check the lockfile version; defaults change.

| Stack | Setting | Default |
|---|---|---|
| Go `database/sql` (also under GORM, sqlx, Ent) | `SetMaxOpenConns` / `SetMaxIdleConns` / `SetConnMaxLifetime` / `SetConnMaxIdleTime` | unlimited / 2 / no limit / no limit. There is no acquire timeout; a caller waits until its context ends |
| Go `pgxpool` (v4, v5) | `MaxConns` / `MinConns` / `MaxConnLifetime` / `MaxConnIdleTime` | max(4, NumCPU) / 0 / 1 h / 30 min |
| HikariCP (Spring Boot default) | `maximumPoolSize` / `minimumIdle` / `connectionTimeout` / `maxLifetime` / `leakDetectionThreshold` | 10 / same as max / 30 s / 30 min / off |
| node-postgres `Pool` | `max` / `idleTimeoutMillis` / `connectionTimeoutMillis` | 10 / 10 s / 0 (waits forever) |
| mysql2 `createPool` | `connectionLimit` / `queueLimit` | 10 / 0 (unlimited queue) |
| Prisma (built-in query engine) | `connection_limit` / `pool_timeout` in the URL | physical CPUs x 2 + 1 / 10 s. With a driver adapter, the adapter's pool settings apply instead |
| Knex | `pool.min` / `pool.max` | 2 / 10 |
| Sequelize | `pool.max` / `min` / `acquire` / `idle` | 5 / 0 / 60 s / 10 s |
| TypeORM | pool comes from the driver | pg: 10; mysql2: 10 |
| SQLAlchemy `QueuePool` | `pool_size` / `max_overflow` / `pool_timeout` / `pool_recycle` | 5 / 10 / 30 s / off |
| Django | `CONN_MAX_AGE` | 0: a new connection per request. Django 5.1+ supports a psycopg 3 pool via `OPTIONS["pool"]` |
| asyncpg `create_pool` | `min_size` / `max_size` | 10 / 10 |
| Rails | `pool` / `checkout_timeout` in `database.yml` | generated default `RAILS_MAX_THREADS` or 5 / 5 s |
| ADO.NET SqlClient, Npgsql | `Max Pool Size` / `Min Pool Size` | 100 / 0 |
| MongoDB drivers | `maxPoolSize` / `minPoolSize` | 100 / 0 |
| PHP-FPM (Laravel, Symfony, WordPress) | no shared pool | about one connection per busy worker process |

Server-side limits: PostgreSQL `max_connections` defaults to 100 (with 3 reserved for
superusers); MySQL `max_connections` defaults to 151. Managed services (RDS, Cloud SQL, Azure)
derive their limit from instance size, so it is `UNKNOWN` unless infrastructure code sets it.

## 3. Sizing arithmetic without inventing workload

Two comparisons are possible from repository evidence alone.

**Demand vs server capacity.**

```text
demand   = sum over deployables of (instances x processes per instance x pool max)
capacity = server max_connections - reserved - other clients (admin tools, migrations, other apps)
```

Fill every factor with a value and its evidence, or `UNKNOWN`. If in-repo values give
`demand > capacity`, the arithmetic is `CONFIRMED` for those values, and the consequence
("connection errors once all pools fill", PostgreSQL `too many clients already`, MySQL
`Too many connections`) is `INFERRED`. Production values may differ; say so. If any factor is
`UNKNOWN`, report the comparison as conditional.

**Pool vs application concurrency.** Compare the pool maximum with what can ask for a connection
at once: HTTP server threads (Spring Boot's embedded Tomcat allows 200 threads by default
against Hikari's 10), Puma threads, gunicorn threads, async concurrency, per-request fan-out
(`Promise.all`, `errgroup`), and background jobs sharing the pool. Excess callers queue for a
connection. That is not automatically wrong, because a small pool is often the right size. It
becomes a finding when combined with long holders (section 4) or no acquire timeout.

What the repository cannot tell you: whether the pool is too small or too large. Write
"Cannot determine whether the pool size is appropriate without runtime metrics (pool wait time,
connections in use, database CPU and active sessions)." Never recommend increasing the pool
without evidence of acquire waits **and** database headroom. More connections can lower database
throughput through contention.

**Idle and lifetime settings.**

- Go `database/sql` with a large `MaxOpenConns` and the default `MaxIdleConns` of 2: after a
  burst, connections above 2 are closed when returned, so the next burst reconnects (TCP, TLS,
  authentication). `INFERRED` churn; `CONFIRMED` configuration.
- No maximum lifetime: connections are never recycled. That can matter behind load balancers,
  proxies with idle timeouts, or DNS-based failover (`INFERRED`, environment-dependent).

## 4. What holds a connection

Each of these keeps a connection checked out, reducing the effective pool for everyone else:

| Holder | Duration |
|---|---|
| An open transaction | until commit or rollback, including remote calls inside it (`transaction-performance.md`) |
| An interactive ORM transaction (Prisma interactive `$transaction`: default `maxWait` 2 s, `timeout` 5 s) | the callback's full duration |
| Iterating a result set (`rows.Next()`, a server-side cursor, a stream) | until the rows are closed |
| Session features: `LISTEN`, session-level advisory locks, temporary tables | the session's life |
| A long report or export on the same pool as request traffic | the query's duration |
| Per-request fan-out | up to N connections at once for one request |
| Open-session-in-view (Spring) | potentially the whole request |

## 5. Connection and cursor leaks

A leak holds a connection until something external ends it. Leaks that happen on a normal code
path eventually exhaust the pool for **every** caller, which is why they rank high even when the
path is infrequent.

**Go `database/sql`**

| Pattern | Effect |
|---|---|
| `rows` from `Query`/`QueryContext` not closed on every path: a `return` or `break` inside `for rows.Next()`, or an early return with no `defer rows.Close()` | the connection stays checked out. A context passed to `QueryContext` closes the rows when it is canceled, so a leak with the request context lasts until the request ends; with `Query` (no context) or `context.Background()`, nothing closes it |
| `db.Query` used for a statement whose result is discarded (`db.Query("UPDATE ...")`) | the returned `Rows` is never closed; use `Exec` |
| `defer rows.Close()` inside a loop | closes only when the function returns; holds one connection per iteration |
| `rows.Err()` not checked after the loop | errors in the middle of iteration truncate results silently (correctness) |
| `db.Conn(ctx)` without `conn.Close()`; `tx` without `Commit`/`Rollback` on some path | leaked connection |
| Not a leak | `QueryRow(...).Scan(...)` closes itself; iterating until `Next()` returns false closes the rows |

**pgx native**: `rows.Close()` or `pgx.CollectRows`; `pool.Acquire` needs `conn.Release()`.
**Node**: `pool.connect()` needs `client.release()` in `finally`; `pool.query` releases by
itself. mysql2 `getConnection` needs `release()`. With node-postgres' default
`connectionTimeoutMillis: 0`, callers wait forever once the pool is exhausted.
**JVM**: JDBC `Connection`, `Statement`, `ResultSet` outside try-with-resources; Hikari
`leakDetectionThreshold` reports them.
**Python**: `pool.getconn()` without `putconn`; a SQLAlchemy `Session` not closed at the end of
a request or task.
**Ruby**: threads outside the request cycle that use ActiveRecord without `with_connection`
hold a connection for the thread's life.
**.NET**: connections not disposed (`using`).

Report a leak as the `CONFIRMED` code pattern plus the `INFERRED` consequence, and give the
path and its frequency class.

## 6. Context, cancellation, and timeouts

For each important database call, record which bounds apply. Do not prescribe values.

| Bound | Where to look |
|---|---|
| Caller cancellation | Go: `QueryContext(r.Context())` versus `Query` or `context.Background()` in jobs; Node and Python: usually none unless the driver supports cancellation |
| Statement timeout on the server | PostgreSQL `statement_timeout`, `lock_timeout`, `idle_in_transaction_session_timeout` (in the DSN `options=-c ...`, `SET`, or role/database settings, which live outside the repository and are `UNKNOWN`); MySQL `max_execution_time` (SELECT only), `innodb_lock_wait_timeout`, `wait_timeout` |
| Client-side statement timeout | JDBC `setQueryTimeout`, Hibernate query timeout hint, `@Transactional(timeout)`, node-postgres `statement_timeout`/`query_timeout`, Knex `.timeout()` |
| Pool acquire timeout | table in section 2; `database/sql` has none beyond the context |
| Transaction timeout | Prisma interactive transactions; Spring `@Transactional(timeout)` |
| Upstream request timeout | HTTP server write timeout, load balancer or gateway timeout (usually outside the repository) |

A database call with no bound at any layer is `CONFIRMED` for the layers you inspected; bounds
set outside the repository stay `UNKNOWN`. Whether a client-side cancellation also stops the
query on the server depends on the driver (a PostgreSQL cancel request, or closing the
connection) and its version; record it as `INFERRED` with the driver named. A client that only
stops waiting leaves the query running on the server, holding its locks and its backend.

## 7. Poolers and proxies

Detect them from DSNs (PgBouncer conventionally listens on port 6432; Prisma
`pgbouncer=true`), compose services, or Helm charts.

- **PgBouncer transaction pooling** does not preserve session state across transactions:
  session `SET`, session advisory locks, `LISTEN`, temporary tables, and prepared statements.
  pgx caches prepared statements by default, which breaks under transaction pooling unless the
  exec mode is changed or PgBouncer 1.21+ `max_prepared_statements` is set.
- **RDS Proxy** pins a client to a connection when it uses session state, which reduces
  multiplexing.
- **Serverless functions** open a pool per concurrent instance, so demand scales with
  concurrency.

## 8. Validation

- Pool metrics: Go `db.Stats()` (`InUse`, `Idle`, `WaitCount`, `WaitDuration`); pgxpool `Stat()`;
  Hikari `hikaricp_connections_active`, `_pending`, `_acquire`; node-postgres `totalCount`,
  `idleCount`, `waitingCount`; SQLAlchemy `pool.status()`.
- Server: PostgreSQL `pg_stat_activity` grouped by `state` and `application_name`; MySQL
  `SHOW PROCESSLIST`, `Threads_connected`, `Max_used_connections`.
- A leak shows as in-use connections that climb and stay up while traffic is flat, and as
  `idle` (PostgreSQL) sessions that belong to the app but are never reused. Reproduce by calling
  the suspected path repeatedly in a test and watching `InUse`.
