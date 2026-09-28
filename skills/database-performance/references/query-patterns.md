# Query Patterns

How to recognize the query patterns that make database cost grow: N+1, queries in loops, scans,
over-fetching, and expensive clauses. Also covers how to tell a real pattern from one that only
looks like it. Every pattern here is a claim about code structure first (`CONFIRMED` when you
read it) and a claim about cost second (`INFERRED` unless measured).

## Contents

1. Amplification notation
2. N+1
3. Queries in loops, reads and writes
4. Scan risk: unbounded and non-selective queries
5. SELECT * and large payloads
6. COUNT vs EXISTS
7. DISTINCT, GROUP BY, ORDER BY
8. JOINs
9. Subqueries and CTEs
10. IN lists, OR, and dynamic filters
11. Looks like a problem, usually is not

## 1. Amplification notation

Quantify structure, never time. Name every variable and what bounds it.

```text
1 request  GET /accounts/{id}/invoices
  -> 1   invoices page           InvoiceRepo.ListByAccount      LIMIT page_size
  -> 1   invoices count          InvoiceRepo.CountByAccount
  -> N   lines of each invoice   LineRepo.ByInvoice             inside for-range over invoices
  -> N   payer of each invoice   AccountRepo.Get                inside the same loop
= 2 + 2N statements per request
  N = invoices on the page; N <= page_size <= 100 (clamp in InvoiceHandler.List)   CONFIRMED
```

- Nested iteration multiplies: `N x M`. Write both variables.
- If nothing bounds N (request body arrays, whole-table iteration), write
  `N = <what> (unbounded)`.
- Apply the same notation to writes (statements per logical operation), to transactions
  (statements and row locks per transaction), and to jobs (statements per run, times the number
  of instances that run it).
- Concurrent fan-out (`Promise.all`, `errgroup`, goroutines) keeps the statement count but runs
  statements in parallel: note that it can occupy up to `min(N, pool size)` connections at once.

## 2. N+1

Call something N+1 only when **all four** are established:

1. A parent query returns a collection. (`CONFIRMED` from the code.)
2. Code iterates over that collection: `for`, `range`, `map`, `forEach`, list comprehension,
   `Promise.all(items.map(...))`, a template loop, a serializer, or a per-parent GraphQL field
   resolver.
3. A query executes for each element: a direct call, a repository method you traced, or an ORM
   lazy load (`orm-performance.md`).
4. No mechanism batches the per-element queries: DataLoader, Prisma's same-tick `findUnique`
   batching, Hibernate `@BatchSize` or `default_batch_fetch_size`, ORM eager loading on the
   parent query.

| What you established | Label |
|---|---|
| 1 to 4 read in code | `CONFIRMED` N+1 |
| 1 and 2 read; 3 depends on ORM lazy-loading behavior | `INFERRED` N+1, with the ORM rule and version |
| 1 to 3 read; a batching mechanism may apply | not N+1 until you check it; if you cannot, `UNKNOWN`, and verify with a query log |
| Several different queries in one handler, no iteration | not N+1 (at most "sequential round trips", usually not a finding) |

Record: parent query, iteration site, child query, what N is and what bounds it, the total
(`1 + N`, `1 + 2N`), and the replacement options:

- batch load the children with `IN (...)` or `= ANY($1)`, then group in memory
- a join, when the child side is to-one or small
- ORM eager loading (`Preload`, `include`, `select_related`, `prefetch_related`, `JOIN FETCH`,
  `with()`)
- a request-scoped loader (DataLoader) for GraphQL and serializer trees

Trade-offs to state: IN-list size limits and memory for large N; row duplication when joining a
to-many relation; a Cartesian product when one query eagerly loads two independent collections;
loss of the per-item error path.

**Where the iteration hides.** Serializers (DRF nested serializers and `SerializerMethodField`,
Rails `as_json(include:)`, Jackson serializing lazy collections), templates (`{% for %}` plus
`item.owner.name`), GraphQL resolvers (one call per parent unless a loader batches them), admin
list views, and batch message consumers that handle each message separately.

**Sequential vs concurrent N+1.** Sequential N+1 adds N dependent round trips to the request.
Concurrent N+1 shortens the wall-clock time but can take up to N pool connections at once and
queue other requests behind them. Both are the same statement count.

## 3. Queries in loops, reads and writes

A query inside a loop over runtime data is a finding even when it is not a parent/child N+1,
for example a lookup repeated per item, or a write per item.

| Pattern in the loop | Statements | Alternatives (check the engine and repository conventions) |
|---|---|---|
| `INSERT` per item | N | multi-row `INSERT`; PostgreSQL `COPY` (pgx `CopyFrom`, psycopg `copy`); `INSERT ... SELECT FROM unnest(...)`; ORM bulk APIs (`pagination.md` section 7) |
| `UPDATE` per item, same new value | N | one `UPDATE ... WHERE id = ANY($1)` or `IN (...)` |
| `UPDATE` per item, different values | N | PostgreSQL `UPDATE ... FROM (VALUES ...)` or `FROM unnest($1::bigint[], $2::int[])`; MySQL `INSERT ... ON DUPLICATE KEY UPDATE` (upsert semantics differ) or `CASE`; pipelined batches (pgx `SendBatch`, JDBC batches) save round trips but not statements |
| `DELETE` per item | N | `DELETE ... WHERE id = ANY(...)`; delete in bounded chunks for large volumes |
| read, modify in code, write back | 2N, plus a lost-update race | one set-based `UPDATE` with an expression |
| the same lookup on every iteration (config, current user, a parent) | N | load once before the loop |
| every write in its own autocommit | N commits | one transaction or chunked transactions (`transaction-performance.md` section 5) |

**Correctness first.** Batching changes behavior. One bad row fails the whole batch instead of
one item. Bulk ORM APIs usually skip per-row hooks, callbacks, validations, and signals
(Django `bulk_create`, Rails `insert_all`). Triggers still fire per row. The order of returned
IDs may not match the input order. State every change of this kind next to the alternative.

## 4. Scan risk: unbounded and non-selective queries

A full scan is a problem only when the table is large or growing, the query is frequent, and
the filter is selective enough that an index would read much less. Decide with what the code
shows, and write the rest as conditions:

| Question | Evidence | If unknown |
|---|---|---|
| Is there a filter? | the SQL | |
| Can an index serve it? | `indexing.md` sections 1 to 3 | |
| Is the result bounded? | `LIMIT`, clamp, pagination | "unbounded" is `CONFIRMED` |
| How often does it run? | frequency class | `UNKNOWN` |
| How large is the table, and does it grow? | growth signals, runtime evidence | "size `UNKNOWN`; risk is conditional on <table> being large" |
| How selective is the filter? | data distribution | `UNKNOWN`; do not guess percentages |

**Unbounded reads** load every matching row into memory: Go slices from `rows.Next()` loops,
`sqlx.Select`, `findMany` without `take`, `list(queryset)`, `getResultList()`, `.all`. On a
request path over a growing table, this is a finding even with an index, because the result
grows with the data (cost driver `KEY_FANOUT` or `TABLE`).

A full scan of a lookup table (countries, feature flags, currencies) is not a finding.

## 5. SELECT * and large payloads

Report `SELECT *` (and ORM calls that load every column) only when it costs something:

1. List the table's columns from the effective schema. Flag large types: `TEXT`, `JSON`,
   `JSONB`, `BYTEA`, `BLOB`, `LONGTEXT`, arrays, embedded documents.
2. Find what the caller uses: the response DTO, the serializer, the fields read afterwards.
3. It is a finding if large columns are fetched and not used, **and** the query returns many
   rows or runs frequently. Otherwise it is at most an `OBSERVATION`.

Mechanics worth knowing (`INFERRED` when relied on): PostgreSQL stores large values out of line
(TOAST) and reads them only when the column is selected, so selecting them has a real read cost;
InnoDB stores long `TEXT`/`BLOB` values off-page similarly. `SELECT *` also prevents
index-only scans. With positional scanning (Go `rows.Scan`), `SELECT *` breaks when a column is
added: a correctness note, not a performance one.

Never state an actual payload size without measurement. Say "column type allows values up to
<limit>; actual size `UNKNOWN`".

## 6. COUNT vs EXISTS

| Code uses the count for | Replace with an existence check? |
|---|---|
| `> 0`, `== 0`, `!= 0`, truthiness | yes: `EXISTS (...)` or `SELECT 1 ... LIMIT 1` stops at the first match; `COUNT` visits every matching row (neither PostgreSQL nor InnoDB stores a row count) |
| a pagination total or page count | no: the number is shown. Options: `LIMIT page_size + 1` for "has more", an estimate, a capped count (`SELECT count(*) FROM (SELECT 1 ... LIMIT 1001) t`), or a cached count; each changes the UX |
| a dashboard or report figure | no: consider whether an approximate or cached value is acceptable |
| a threshold (`count >= 5`) | a capped count (`LIMIT 5`) |

ORM equivalents: Django `.exists()` vs `.count()` vs `len(qs)` (loads all rows); Rails `exists?`
and `any?` (issue `SELECT 1 LIMIT 1` on an unloaded relation) vs `present?`/`blank?` (load all
records) vs `count`/`size`/`length`; Spring Data `existsBy...` vs `countBy...`; Prisma
`findFirst({ select: { id: true } })` vs `count`; GORM `Limit(1).Find` vs `Count`.

`COUNT(column)` skips NULLs, so it is not interchangeable with `COUNT(*)`.

## 7. DISTINCT, GROUP BY, ORDER BY

- **`DISTINCT` after a join** often removes duplicates created by joining a to-many relation.
  If the query only filters by the child, an `EXISTS` semi-join returns the same parent rows
  without producing and removing the duplicates. It is equivalent only when every selected
  column comes from the parent.
- **`GROUP BY` and aggregates** over large inputs need memory; when it runs out, PostgreSQL
  spills (`external merge`, hash aggregate batches) and MySQL uses on-disk temporary tables. An
  index on the grouping columns can let the database aggregate in order. Whole-table aggregates
  on request paths are a `TABLE` cost driver.
- **`ORDER BY` without a supporting index** sorts every matching row. With `LIMIT`, that is
  still a read of every matching row (`indexing.md` section 1).
- **`ORDER BY RANDOM()` / `RAND()`** reads and sorts the whole input.
- **`ORDER BY` on a non-unique column with `LIMIT`/`OFFSET`** gives a nondeterministic order
  among ties, so pages can repeat or skip rows. This is a correctness finding; add a unique
  tie-breaker.

Describe the potential cost, why it arises, and how to verify it. Do not claim the clause is
slow.

## 8. JOINs

For each join in an important query, check:

| Check | Risk |
|---|---|
| Is the join column on the looked-up side indexed? | a scan per lookup; in PostgreSQL, SQLite, and SQL Server, FK columns are not indexed automatically |
| Do the join columns have the same type, charset, and collation? | the index cannot be used for the comparison (a classic MySQL problem) |
| Cardinality: is either side to-many? | the row count multiplies |
| Are two independent to-many relations joined to the same parent? | a Cartesian product: `rows = children_a x children_b` per parent |
| Is there a `CROSS JOIN`, or a comma join without a join predicate? | a Cartesian product, `CONFIRMED` from the SQL |
| `LEFT JOIN` with a `WHERE` condition on the right table | it becomes an inner join: a correctness issue, often unintended |
| More than about 8 joined relations (PostgreSQL) | the planner stops searching every join order (`join_collapse_limit`); plan quality may vary |

Do not state row counts or join algorithms without a plan. The optimizer chooses join order and
method; static analysis can only show which choices are available (for example, "no index on
`invoice_lines.invoice_id`, so a nested loop from `invoices` would scan `invoice_lines` per
invoice; a hash join avoids that; plan not available").

## 9. Subqueries and CTEs

- **Correlated scalar subquery in the `SELECT` list** (`SELECT ..., (SELECT count(*) FROM lines
  l WHERE l.invoice_id = i.id) FROM invoices i`) is evaluated per outer row unless the optimizer
  rewrites it. In a PostgreSQL plan it appears as `SubPlan` with loops equal to the outer rows.
  Needs an index on the correlated column; an aggregate join or `LATERAL` may be an alternative.
- **`EXISTS` / `IN (subquery)` in `WHERE`** are usually turned into semi-joins. They are
  generally not a problem by themselves; check that the correlated column is indexed.
- **`NOT IN (subquery)`** returns no rows when the subquery yields a NULL, and in PostgreSQL it
  cannot be planned as an anti-join. `NOT EXISTS` is the usual correct form. This is correctness
  first, performance second.
- **CTEs.** PostgreSQL 11 and earlier always materialize a CTE (an optimization fence).
  PostgreSQL 12+ inlines a non-recursive, side-effect-free CTE referenced once, unless it is
  declared `MATERIALIZED`. MySQL 8.0+ may merge or materialize a derived table or CTE. The
  engine version decides; state it.
- **Recursive CTEs.** Check the termination condition and a depth bound. Cyclic data without a
  guard (PostgreSQL 14+ `CYCLE` clause, or a visited array) can run until it hits a limit.
- **Window functions** sort by their `PARTITION BY` and `ORDER BY`; `ROW_NUMBER() ... WHERE rn =
  1` for "latest per group" reads every row of every group. PostgreSQL alternatives are
  `DISTINCT ON`, or `LATERAL (... ORDER BY ... LIMIT 1)` with an index. Verify with a plan.

Do not rewrite a valid query in the report. Explain the consideration and how to check it.

## 10. IN lists, OR, and dynamic filters

- **IN-list size** comes from somewhere: a request array, a previous query, a whole table. Say
  what bounds it. Very large lists cost parse and plan time and hit parameter limits:
  PostgreSQL's protocol allows 65,535 bind parameters per statement, SQL Server 2,100, SQLite
  32,766 by default since 3.32 (999 before), and MySQL is limited by `max_allowed_packet`. In
  PostgreSQL, one array parameter (`= ANY($1)`) avoids the parameter limit.
- **`OR` across columns**: see `indexing.md` section 3.
- **Optional filters** built conditionally produce several query shapes; each needs its own
  index consideration. List the shapes that production paths use.
- **User-selected sort columns**: check for an allowlist (otherwise it is also an injection
  question) and whether each allowed column is indexed with the filters it combines with.

## 11. Looks like a problem, usually is not

| Pattern | Why it is usually fine | Report only if |
|---|---|---|
| Several queries in one handler | different queries in sequence are normal | they are in a loop, or the same query repeats |
| Loop over a small fixed constant list | the cost is constant | the constant is large, or each query is expensive (then usually `LOW`) |
| Children loaded with `IN`/`ANY` | that is the batched form | the IN list is unbounded |
| ORM eager loading (`include`, `Preload`, `prefetch_related`) | a query per relation, not per row | it loads unbounded children, or joins two to-many relations |
| `SELECT *` on a narrow table | nothing large is fetched | large columns are unused on a hot path |
| Full scan of a small lookup table | cheap and correct | the table grows |
| `COUNT` for a pagination total | the count is needed | the cost grows with matches and is on a hot path |
| `OFFSET` with a bounded maximum page | the work is bounded | the page is unbounded or crawled |
| A repository method with no caller | it never runs | documentation or configuration says it runs |
| A query without an index on a `RARE` path | the index would cost writes on hot paths | the table is large and the rare path has a latency requirement |
