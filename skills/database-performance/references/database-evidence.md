# Database Evidence

How to record runtime and measured evidence, read execution plans and database statistics that
the user supplies or that you are able to capture, and write validation plans that separate
expected improvement from measured improvement. The evidence bases (`STATIC`, `RUNTIME`,
`MEASURED`) and labels are defined in `SKILL.md`; this reference applies them.

## Contents

1. Recording evidence
2. Deriving numbers from measurements
3. Reading PostgreSQL plans
4. Reading MySQL and MariaDB plans
5. SQLite, SQL Server, MongoDB
6. Workload statistics
7. Capturing query counts
8. Pitfalls when validating
9. Validation plan
10. Expected vs measured improvement

## 1. Recording evidence

**RUNTIME** evidence records what happened in a running system: which statements ran and how
many, a plan's shape, a setting's live value, table sizes from the catalog, an error in a log.

```text
RUNTIME  GET /accounts/{id}/invoices issued 52 statements for page_size=50
  source       SQL log captured by the user in staging, file logs/invoices-sql.txt
  captured     2026-03-02
```

**MEASURED** evidence is a number for cost or load, with its source and conditions:

```text
MEASURED  p95 latency of GET /accounts/{id}/invoices = 850 ms
  source       APM dashboard screenshot supplied by the user
  scope        endpoint        environment  production
  window       2026-02-24 to 2026-03-02     method  APM request traces
```

Rules:

- A number without a source is not evidence. If the user gives a number without source,
  window, or method, record it as `MEASURED` with `source: user report` and write the missing
  fields as `UNKNOWN`. Do not drop it, and do not treat it as more precise than it is.
- Keep the conditions with the number. A plan from a laptop with 1,000 rows says little about
  production with an `UNKNOWN` number of rows.
- Runtime evidence can contradict static reasoning. When it does, the runtime evidence wins,
  and the report says which static claim it overturned.
- Never produce latency, QPS, row counts, CPU, IOPS, lock waits, cache hit rates, pool
  utilization, or execution times that no measurement contains. The phrases "typically",
  "usually around", "a few milliseconds", and "for a table of 1M rows" introduce invented
  numbers. Use the growth statement instead: "cost grows with the number of the account's
  invoices".

## 2. Deriving numbers from measurements

Arithmetic on measured values is allowed only when every input is `MEASURED` or `RUNTIME` with
a source. The result is `INFERRED`, with the formula shown and its assumption named:

```text
INFERRED  about 50 x 1.2 ms = 60 ms of sequential round trips per request
  inputs   N = 50 (RUNTIME, SQL log); 1.2 ms mean for the child query (MEASURED, pg_stat_statements)
  assumes  round trips are sequential and the mean holds for this endpoint
```

Without measured inputs there is no number, only the structure: `1 + N sequential round
trips`.

## 3. Reading PostgreSQL plans

- `EXPLAIN` shows the plan and **estimates** (cost units, rows). It is `RUNTIME` evidence of
  the plan's shape, not of time.
- `EXPLAIN (ANALYZE, BUFFERS)` **executes** the statement and reports actual time, rows, loops,
  and buffers: `MEASURED`. For data-modifying statements, wrap it in `BEGIN; ... ROLLBACK;`. Do
  not run it on a production primary without the owner's agreement.
- `EXPLAIN (GENERIC_PLAN)` (PostgreSQL 16+) shows the plan for a parameterized statement without
  values.

| Node | Meaning | Look for |
|---|---|---|
| Seq Scan | reads the whole table | correct for small tables or when most rows match; "Rows Removed by Filter" far above the rows kept suggests a missing index |
| Index Scan | walks the index, fetches rows from the table | whether the index condition covers the filter, or a filter runs after |
| Index Only Scan | answers from the index | "Heap Fetches" well above 0 means the visibility map is stale (vacuum) |
| Bitmap Index / Bitmap Heap Scan | collects row locations, then reads pages in order | "Recheck Cond", "lossy" blocks when `work_mem` is small; BitmapAnd/Or combine indexes |
| Nested Loop | runs the inner side once per outer row | inner time and rows are **per loop**: multiply by `loops` |
| Hash Join | builds a hash of one side | "Batches" above 1 means it spilled to disk |
| Merge Join | merges two sorted inputs | the sorts feeding it |
| Sort | sorts its input | "Sort Method: external merge  Disk" spilled; "top-N heapsort" with LIMIT; a sort under a Limit means the index did not provide the order |
| HashAggregate / GroupAggregate | aggregation | disk batches in newer versions |
| SubPlan | a subquery run per outer row | its loops |
| Memoize (14+), Gather (parallel), CTE Scan, Materialize | caching, parallelism, CTEs | |

Signals: estimated rows off from actual rows by an order of magnitude or more (stale or missing
statistics: `ANALYZE`, extended statistics); `shared read` far above `shared hit` (data read from
outside the buffer cache); planning time comparable to execution time; JIT time dominating
short queries.

Prepared statements: after five executions, PostgreSQL may switch to a generic plan that ignores
the parameter values (`plan_cache_mode`). A plan from `EXPLAIN` with literal values can differ
from the plan the application gets. `auto_explain` (`log_min_duration`, `log_analyze`) logs the
plans the application actually runs.

## 4. Reading MySQL and MariaDB plans

`EXPLAIN` columns, from best to worst `type`: `system`, `const`, `eq_ref`, `ref`, `range`,
`index` (a full index scan), `ALL` (a full table scan). `key` is the chosen index; `key_len`
shows how many index columns are used; `rows` and `filtered` are estimates. `Extra`: `Using
index` (covering), `Using where`, `Using index condition` (index condition pushdown), `Using
filesort` (a sort not served by an index), `Using temporary` (a temporary table).

- `EXPLAIN FORMAT=TREE` (8.0.16+) shows the iterator tree; `EXPLAIN FORMAT=JSON` shows costs.
- `EXPLAIN ANALYZE` (8.0.18+) executes the statement and reports actual time and rows:
  `MEASURED`. MariaDB uses the `ANALYZE <statement>` form instead.
- The optimizer trace (`optimizer_trace`) explains why a plan was chosen.

## 5. SQLite, SQL Server, MongoDB

- **SQLite**: `EXPLAIN QUERY PLAN`: `SCAN t` (full scan), `SEARCH t USING INDEX ...`, `USING
  COVERING INDEX`, `USE TEMP B-TREE FOR ORDER BY` (a sort). `ANALYZE` populates `sqlite_stat1`
  for the planner.
- **SQL Server**: actual execution plans (`SET STATISTICS XML ON`), `SET STATISTICS IO, TIME ON`
  (logical reads), Query Store runtime statistics. Look for Index Seek vs Scan, Key Lookups,
  and spill warnings. Missing-index DMVs are suggestions, not proof.
- **MongoDB**: `explain("executionStats")`: `COLLSCAN` vs `IXSCAN`, `FETCH`, an in-memory
  `SORT`; compare `totalKeysExamined` and `totalDocsExamined` with `nReturned`. The profiler and
  `$indexStats` give workload data.

## 6. Workload statistics

| Question | PostgreSQL | MySQL |
|---|---|---|
| Which statements cost the most in total | `pg_stat_statements`: `calls`, `total_exec_time`, `mean_exec_time`, `rows`, `shared_blks_hit`, `shared_blks_read` (PostgreSQL 13+ names; `total_time`/`mean_time` before 13) | `performance_schema.events_statements_summary_by_digest`: `COUNT_STAR`, `SUM_TIMER_WAIT` (picoseconds), `SUM_ROWS_EXAMINED`, `SUM_ROWS_SENT`, `SUM_NO_INDEX_USED` |
| Slow individual statements | `log_min_duration_statement`, `auto_explain` | slow query log (`long_query_time`), `sys.statements_with_full_table_scans`, `sys.statements_with_sorting` |
| Scans per table | `pg_stat_user_tables`: `seq_scan`, `seq_tup_read`, `idx_scan`, `n_live_tup`, `n_dead_tup` | `sys.schema_table_statistics` |
| Index use | `pg_stat_user_indexes.idx_scan` | `sys.schema_unused_indexes`, `sys.schema_redundant_indexes` |
| Table size | `pg_class.reltuples` (estimate), `pg_total_relation_size()` | `information_schema.TABLES` (`TABLE_ROWS` is an estimate for InnoDB) |
| Live sessions, waits | `pg_stat_activity`, `pg_locks` | `SHOW PROCESSLIST`, `performance_schema.data_lock_waits` |

Statistics are cumulative since the last reset, and per server (replicas keep their own). Record
the reset time or the window.

## 7. Capturing query counts

Query count per request or per job run turns an `INFERRED` N+1 into `RUNTIME` evidence cheaply:

- ORM or driver SQL logging during one request (`orm-performance.md` section 2)
- a test that asserts the count (Django `assertNumQueries`, a Prisma query-event counter, a
  Hibernate statistics check), run with realistic N
- the `pg_stat_statements.calls` delta around a single request in an isolated environment
- database spans per trace in APM (OpenTelemetry, Datadog, New Relic)

## 8. Pitfalls when validating

- **Data volume.** On small tables the planner correctly prefers full scans, so development
  plans hide index problems. Use production-like volume and distribution, and state it.
- **Cache state.** The first run reads from disk; later runs from memory. Report warm and cold,
  or discard warm-up runs, and say which.
- **Instrumentation overhead.** `EXPLAIN ANALYZE` timing adds overhead; compare relative
  results, and use `TIMING OFF` when only row counts matter.
- **Parameters.** Literal values and prepared statements can get different plans (section 3).
- **Statistics freshness.** After bulk loading test data, run `ANALYZE` before comparing plans.
- **Concurrency.** Single-query timings do not show lock waits or pool waits. Contention needs a
  concurrent load test.
- **Tails.** Averages hide tails. Use percentiles, and report the sample size.
- **One run is not a benchmark.** Repeat, and report the spread.
- **Production safety.** `EXPLAIN ANALYZE` executes the statement. Heavy reads add load; writes
  change data unless rolled back.

## 9. Validation plan

Write one entry for each significant finding and each recommended change:

```text
V3  for F3 (invoices listed without an index for the sort)
  Hypothesis   the list query reads and sorts all of the account's invoices on every call
  Metric       Sort node present or absent; rows read; shared buffers; endpoint p95
  Baseline     UNKNOWN: capture before any change
  Method       EXPLAIN (ANALYZE, BUFFERS) for an account with many invoices, on staging data
               of production volume; repeat 5 times warm; then APM p95 over one week
  Environment  staging with production-like volume (current volume UNKNOWN)
  Success      no Sort node; rows read bounded by the page size; p95 does not regress for
               writes to invoices
  Status       EXPECTED improvement: not measured
```

## 10. Expected vs measured improvement

| Term | Meaning | Allowed content |
|---|---|---|
| Expected improvement | what should change and by what mechanism | direction and mechanism only: "statements per request drop from 1 + N to 2", "the sort disappears", "reads are bounded by LIMIT instead of the account's history". No numbers unless derived under section 2 |
| Measured improvement | what changed | before and after measurements, under comparable conditions, each with a measurement record |

When someone asks for estimated gains before any measurement (for prioritization, or for a
manager), give the mechanism, the cost driver, the priority with its evidence, and the
validation plan that will produce the number. Say plainly that the repository cannot supply the
number.
