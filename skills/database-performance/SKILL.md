---
name: database-performance
description: Use when asked to analyze, review, or explain database performance in a codebase - slow queries or slow endpoints and jobs where the database may be involved, N+1 queries, queries in loops, missing or redundant indexes, full scans, OFFSET pagination, COUNT vs EXISTS, long or lock-heavy transactions, deadlock or contention risk, connection pool sizing, exhaustion, or leaks, database timeouts, bulk writes, ORM query behavior (GORM, Ent, sqlc, Prisma, TypeORM, Sequelize, Hibernate/JPA, Django, SQLAlchemy, ActiveRecord, Eloquent, EF Core), or cache stampedes, on PostgreSQL, MySQL, MariaDB, SQLite, SQL Server, or MongoDB.
---

# Database Performance

Find where the application's use of the database can become a bottleneck. Read the source code,
SQL, ORM calls, schema, migrations, and configuration, and connect every query to the execution
path that runs it. Record the result as reusable text: a Markdown artifact (source of truth) and
an HTML rendering of the same knowledge.

**Core principle:** a repository shows what the code asks the database to do, in what
structure, and how often relative to its callers. It does not show how long anything takes.
Report structure as fact, cost as risk, and time only when it was measured.

```text
Source Code        >  Documentation  >  Assumption
Measured Evidence  >  Static Guess
Actual Query       >  ORM Abstraction
Query Behavior     >  Query Appearance
Correctness        >  Optimization
Impact             >  Number of Findings
```

## When to Use

- "Analyze this repository's database performance" / "review our DB usage" / "review indexes"
- "Find slow queries" / "find N+1 queries" / "why is `GET /invoices` slow?"
- "Analyze PostgreSQL performance" (confirm the engine first; never assume it)
- Reviewing a job, consumer, or import that reads or writes a lot
- Runtime evidence the user supplies (EXPLAIN output, `pg_stat_statements`, slow logs, APM
  numbers) makes the analysis stronger, but the skill works from code alone

Not for: tracing a whole flow end to end (use `distributed-flow`), designing a new schema,
tuning a database server with no application code, or running benchmarks. This skill writes
the validation plan; it does not execute load tests.

## Evidence Model

Every claim carries a **label**, the MDW label used by every skill:

| Label | Use when | Must include |
|---|---|---|
| `CONFIRMED` | you observed it directly | its **basis** (below) and the evidence reference |
| `INFERRED` | it follows from confirmed facts plus reasoning: library defaults, optimizer behavior without a plan, consequences of structure | the evidence and the reasoning step |
| `UNKNOWN` | neither the repository nor supplied evidence can decide it | what would resolve it |

A `CONFIRMED` claim also states its **basis**:

| Basis | Establishes | Comes from |
|---|---|---|
| `STATIC` | what the code, SQL, schema, and in-repo configuration do or declare | reading the repository |
| `RUNTIME` | what happened when the system ran: statements executed and their count, a plan's shape, live settings, table sizes | SQL logs, `EXPLAIN` without `ANALYZE`, catalog queries, traces |
| `MEASURED` | how much it cost: time, rows read, buffers, rates, utilization, with source and window | `EXPLAIN ANALYZE`, `pg_stat_statements`, `performance_schema`, APM, benchmarks |

Write `CONFIRMED (STATIC)`, `CONFIRMED (RUNTIME)`, `CONFIRMED (MEASURED)`, `INFERRED`, or
`UNKNOWN`. Rules:

- `STATIC` evidence establishes structure (a query inside a loop, no index on a column, an HTTP
  call between `BEGIN` and `COMMIT`), never speed.
- Without a plan, how the database executes a query is `INFERRED`. Write "Execution plan not
  available. Run `EXPLAIN (ANALYZE, BUFFERS)` to validate." (PostgreSQL) or the engine's
  equivalent (`database-evidence.md`).
- An ORM call is `CONFIRMED`; the SQL it generates is `INFERRED` for that ORM version; the exact
  SQL is `UNKNOWN` until captured.
- An absence is `CONFIRMED` only where you looked: "no index in the migrations" is `CONFIRMED
  (STATIC)`; whether production has one is `UNKNOWN`.
- Runtime evidence outranks static reasoning. When they disagree, say which static claim it
  overturned.
- Documentation, comments, and names are leads. When code disagrees, the code wins and the
  disagreement is **documentation drift**.

**Never invent** production traffic, latency, execution time, database or table size, row
counts, execution plans, CPU, IOPS, lock waits, cache hit rates, pool utilization, or expected
improvement percentages. This rule has no exception for labeled estimates, "rough" figures, or
an assumed reference workload. Where a number would go, state what the cost grows with (the
**cost driver**) and how to measure it. The only numbers allowed are values read from code or
configuration (a statement formula `1 + 2N`, a page-size clamp of 100, a 10 s client timeout,
`3 replicas x 50 = 150` connections) and measurements with their source.

| Instead of | Write |
|---|---|
| "This query is slow" | "This query has a potential performance risk because ..." (unless runtime evidence shows it is slow) |
| "MySQL does a full scan and a filesort" | "No index serves the filter and order (`CONFIRMED (STATIC)`); a scan and sort are likely (`INFERRED`). Execution plan not available." |
| "About 40-80 ms of overhead" | "N + 1 sequential round trips, N <= 20" |
| "Improves latency by 90%" | "Expected: the sort disappears and reads are bounded by LIMIT. Measured: not yet (V3)." |
| "Remove this index" | "`POTENTIALLY REDUNDANT`: ..." |
| "The pool is too small" | "Cannot determine whether the pool size is appropriate without runtime metrics." |
| "This deadlocks" | "Potential deadlock risk: T1 locks A then B while T2 locks B then A." |

## Workflow

Copy this checklist and complete it in order:

```text
[ ] 1. Scope the question
[ ] 2. Discover the repository and the databases
[ ] 3. Build the effective schema
[ ] 4. Discover data access and build the query inventory
[ ] 5. Analyze query patterns
[ ] 6. Analyze indexes
[ ] 7. Analyze transactions
[ ] 8. Analyze concurrency and locking
[ ] 9. Analyze the connection pool, leaks, and timeouts
[ ] 10. Analyze pagination and batch operations
[ ] 11. Analyze cache interaction
[ ] 12. Combine along execution paths
[ ] 13. Assess risk: severity and priority
[ ] 14. Validate the evidence
[ ] 15. Write the Markdown artifact, then the HTML artifact
[ ] 16. Self-check
```

### 1. Scope the question

Decide before reading in depth: what is analyzed (the whole repository, named entry points,
jobs, or tables), which databases are involved, and what runtime evidence exists. If the user
may have plans, statistics, slow logs, or APM data, ask once. Do not block on it. For "why is X
slow?", the answer is a ranked set of **potential contributors** with their evidence, plus the
measurement that would decide between them. Never an asserted cause without runtime evidence.

### 2. Discover the repository and the databases

Identify the engine and version, driver, ORM or query builder (resolved versions from the
lockfile), migration tool, pool construction, poolers or proxies, replicas, caches, and which
deployable uses which database. Two agreeing signals from code or infrastructure beat any
README. Name the engine only from evidence; a PostgreSQL driver proves the protocol, not the
engine. See `references/query-analysis.md` section 1.

### 3. Build the effective schema

Replay every migration in order. Record tables, large columns, keys, foreign keys and whether
they are indexed, unique constraints, indexes (columns, order, type, predicate), soft-delete
columns, partitions, views, and triggers. An index created early and dropped later does not
exist. Migrations and dumped schemas outrank ORM models. See `references/query-analysis.md`
section 2.

### 4. Discover data access and build the query inventory

Find every call site (raw SQL, builders, ORM calls, stored procedures), resolve the SQL that
actually runs, and walk callers up to an entry point. For each important query, record its
location, entry point, tables, **frequency class**, **cost driver**, enclosing loop, and
transaction. A query with no reachable caller is not a risk. See
`references/query-analysis.md` sections 3 to 8 and, for ORM calls,
`references/orm-performance.md`.

### 5. Analyze query patterns

Check N+1 (all four criteria), queries in loops, unbounded reads, scan risk, `SELECT *` with
large columns, `COUNT` used for existence, `DISTINCT` and `GROUP BY` over joins, join
cardinality, correlated subqueries, CTE behavior by engine version, and IN-list bounds. Express
amplification as a formula with every variable named and bounded. See
`references/query-patterns.md`, including its table of patterns that are usually fine.

### 6. Analyze indexes

For each frequent or growing query, compare its access pattern (equality, range, join, order,
group, selected columns, hidden ORM predicates) with the effective indexes, applying leftmost
prefix, range, ordering, backward-scan, and covering rules for this engine. Report candidates in
CANDIDATE form, `POTENTIALLY REDUNDANT` indexes, and unindexed foreign keys where the engine does
not index them. Never generate a migration unless the user asks for one. See
`references/indexing.md`.

### 7. Analyze transactions

Map every transaction on an important path: where it starts and ends, the statements inside
(as a formula), the locks it takes, the non-database work inside (remote calls, sleeps, heavy
CPU), what bounds its duration, and error paths that leave it open. Include implicit
per-request transactions. Report a remote call inside a transaction only after tracing it
between the start and the end. See `references/transaction-performance.md`.

### 8. Analyze concurrency and locking

Inventory explicit and implicit locks with their duration, hot rows, lock order (report
**potential deadlock risk** with the interleaving), table locks and migration DDL, advisory,
in-process, and distributed locks, optimistic retries, and queue tables. See
`references/locking-concurrency.md`.

### 9. Analyze the connection pool, leaks, and timeouts

Record the pool settings per deployable, using library defaults only as `INFERRED` with the
version. Do the sizing arithmetic on in-repo values, with every factor labeled. List what holds
connections, find leaks (rows not closed, transactions not ended, connections not released),
and record which cancellation and timeout bounds apply. Do not prescribe values. See
`references/connection-pool.md`.

### 10. Analyze pagination and batch operations

Identify the pagination style per list path and its bounds (maximum page, page size), totals
queries, ordering stability, whole-table iteration in jobs, and per-item writes that have a bulk
form in this engine and library. See `references/pagination.md`.

### 11. Analyze cache interaction

Inventory each cache (key, TTL, source query, writers, invalidation, coalescing, error
behavior). Report a stampede only when all four conditions hold. Find repeated queries. Do not
recommend a cache for a problem the query itself causes. See `references/caching.md`.

### 12. Combine along execution paths

Findings compound along a path: an N+1 whose child query has no index, a long transaction
holding a hot row, a job running once per replica, a leak on a per-request path. For each entry
point in scope, write the statement tree (`references/report-format.md` section 4) and merge
related observations into one finding with the combined formula (`N x TABLE`,
`replicas x (1 + 2N) per run`). If an MDW `distributed-flow` or `sequence-flow` artifact exists
for the path, use it as a map and verify the hops you rely on.

### 13. Assess risk: severity and priority

**Severity** is the impact category. Take the highest row whose conditions the evidence meets:

| Severity | Conditions |
|---|---|
| `CRITICAL` | Can exhaust or block a shared resource for **every** caller (the connection pool, locks on a shared hot row or table, database-wide load from an unbounded repeated job) on a reachable, normal path, **and** either runtime evidence shows it happening or the mechanism does not depend on unknown workload (for example, a leak on every call of a live path) |
| `HIGH` | Cost grows with runtime data (N, table size, offset depth, unbounded fan-out), or shared resources are held for a duration bounded only by something external (a remote call's timeout), on a per-request, per-message, or frequent scheduled path |
| `MEDIUM` | Growth is real but bounded (a clamped N, a rarely used path, an infrequent job), or the path's frequency is `UNKNOWN` |
| `LOW` | A constant-factor cost with no growth: a fixed number of extra round trips, unused columns on a narrow path |
| `OBSERVATION` | Worth recording, no demonstrated cost: potentially redundant indexes, defaults relied upon, correctness issues found on database paths, trade-offs of the current design |

Append the **qualifier**: `static risk` when the finding rests on `STATIC` and `INFERRED`
evidence only (the severity is provisional; say what runtime evidence would confirm or lower
it), `runtime-observed` with `RUNTIME` evidence, and `measured` with `MEASURED` evidence.
Confidence (whether the pattern exists) and severity (what it could cost) are separate fields.

**Priority** follows from severity and confidence; it is not a score or a schedule:

| Priority | When |
|---|---|
| `IMMEDIATE INVESTIGATION` | `CRITICAL`; or `HIGH` with the pattern `CONFIRMED` on a per-request or per-message path |
| `INVESTIGATE` | any other `HIGH`; `MEDIUM` with the pattern `CONFIRMED` |
| `MONITOR` | `MEDIUM` that is `INFERRED` or conditional on an `UNKNOWN` (size, traffic); `LOW`. Name the metric to watch |
| `OBSERVATION` | `OBSERVATION` |

Every finding states the rule that placed it: "IMMEDIATE INVESTIGATION: HIGH, pattern
CONFIRMED, PER_REQUEST path `GET /accounts/{id}/invoices`".

### 14. Validate the evidence

Before writing, re-check every finding: the location was read; the path is reachable; the
labels and basis are right; the N+1 criteria hold; constant loops and batched loads are not
reported as N+1; nothing is claimed about plans without a plan; every number is a code-derived
count, a configured value, or a measurement with its source; runtime evidence has been applied.
Downgrade or drop what does not survive. Fewer, well-supported findings beat many weak ones.

### 15. Write the artifacts

Read `references/report-format.md` first. It defines section contents, table columns, the
finding block, text diagrams, and the HTML skeleton.

**Output contract**

- Write exactly two files: `<scope>-database-performance.md` and
  `<scope>-database-performance.html`. `<scope>` is a kebab-case name for what was analyzed: the
  repository or service for a whole-repository review, the entry point (`get-accounts-invoices`),
  or the table. The default location is `mdw/database-performance/` at the analyzed repository's
  root; a location the user gives takes precedence. If a file exists, read it first and replace
  it only if it is an earlier MDW artifact for the same scope.
- Markdown is the source of truth. Write it first, then derive the HTML from it. The HTML adds no
  claims and drops none.
- Never produce Mermaid, `.mmd`, `.drawio`, `.svg`, `.png`, `.jpg`, inline `<svg>`, `<canvas>`,
  images, or any other diagram format.
- The HTML is one self-contained file with inline CSS: no framework, no external requests, no
  build step. It opens directly from disk and is readable with JavaScript disabled.
- Do not write migrations, DDL files, patches, or rewritten functions unless the user asks.
  Recommendations describe the change; a short query shape is allowed where it clarifies.

**Markdown structure.** Required sections always appear; the others appear when the scope
contains what they analyze, and Scope lists the omitted ones with the reason.

```md
# Database Performance Analysis: <scope>

## 1. Scope                              (required)
## 2. Database Architecture              (required)
## 3. Query Inventory                    (required)
## 4. Performance Findings               (required)
## 5. Query Analysis
## 6. Index Analysis
## 7. Transaction Analysis
## 8. Concurrency / Locking
## 9. Connection Pool
## 10. Pagination / Batch Operations
## 11. Cache Interaction
## 12. Runtime Evidence                  (required)
## 13. Validation Plan                   (required)
## 14. Recommended Investigations        (required)
## 15. Confirmed / Inferred / Unknown    (required)
## 16. Open Questions
## 17. Evidence                          (required)
```

**Finding block.** Every finding has all of these fields, in this order. A field with nothing
to say gets `none` or `UNKNOWN (resolve by ...)`:

```text
ID and title        F<n>. <pattern>: <what, where>
Severity            <CRITICAL | HIGH | MEDIUM | LOW | OBSERVATION> — <static risk | runtime-observed | measured>
Priority            <IMMEDIATE INVESTIGATION | INVESTIGATE | MONITOR | OBSERVATION> (<rule that placed it>)
Confidence          <label (basis)> for the pattern; <label> for the cost
Location            <file Symbol>
Path                <entry point> -> ... -> <query>
Frequency           <frequency class>
Statements          <formula, with N named and bounded>
Evidence            <file Symbol references and what each shows>
Behavior            <what the code makes the database do>
Potential Impact    <cost driver and conditions; no invented numbers>
How to Verify       <concrete method and metric>
Possible Improvement <repository-specific options>
Trade-offs          <costs, including correctness changes>
Unknowns            <what is not known and how to resolve it>
```

**Scope of findings.** Findings are about the database's behavior. A correctness problem found
on a database path (a lost update, a stale cache, a partial commit) is reported as an
`OBSERVATION` or in the related finding's trade-offs, because correctness outranks
optimization. Issues unrelated to the database (authentication, input validation, framework
error handling) get one line each under Open Questions, **Outside scope**.

Then reply to the user with the two paths and a short summary: the engines, finding counts by
severity, the findings under `IMMEDIATE INVESTIGATION`, whether any runtime evidence was used,
and the number of `UNKNOWN` items. Do not paste the document into the reply.

### 16. Self-check

```text
[ ] Every finding has every field; every CONFIRMED claim names its basis and a file + symbol you read
[ ] No line number you did not read
[ ] Search the Markdown for time, rate, size, and percentage figures (ms, s, %, QPS, rows, GB, MB,
    "~"): each is a code-derived count, a configured value, or has a measurement record
[ ] No "slow", "full scan", "filesort", "Seq Scan" stated as fact without runtime evidence
[ ] Every N+1 meets the four criteria; constant loops and IN/ANY batch loads are not called N+1
[ ] Every index claim uses the effective schema after all migrations, for this engine and version
[ ] No migration or DDL written unless asked; redundant indexes are POTENTIALLY REDUNDANT
[ ] Engine-specific advice matches the engine in evidence (no PostgreSQL advice for MySQL)
[ ] Every recommendation has why, evidence, trade-offs, and a validation entry
[ ] Unreachable code is not reported as a risk; documentation contradicted by code is drift
[ ] Runtime Evidence says "none" when there is none, and severities say "static risk"
[ ] The HTML has the same claims as the Markdown; only .md and .html were written
```

## Rationalizations

| Thought | Reality |
|---|---|
| "The manager needs numbers, so I'll give estimates" | Give the mechanism, cost driver, priority, and the validation plan that will produce the number. Say the repository cannot supply it. |
| "I'll state an assumed reference workload and derive from it" | An assumed workload is invented data. Every number derived from it is invented, however it is labeled. |
| "I'll mark the figures as estimates, not promises" | Readers quote numbers and drop disclaimers. A labeled guess is still a guess. |
| "Without that index the database must scan and sort" | Optimizers choose: small tables, bitmap scans, skip scans, other indexes. Without a plan it is `INFERRED`. |
| "The fix is obvious; I'll write the migration" | A recommendation is not a migration. Use the CANDIDATE block; write DDL only on request. |
| "`(a)` is redundant next to `(a, b)`; drop it" | `POTENTIALLY REDUNDANT`. Removal needs usage statistics from every server over a full business cycle. |
| "Soft delete everywhere, so add partial indexes" | That depends on the deleted fraction (`UNKNOWN`) and exact predicates. Report the dependency. |
| "Three COUNTs in a loop is an N+1" | A loop over a constant list is constant cost. N+1 needs a runtime collection. |
| "The pool looks too small (or too big)" | Only the arithmetic against configured limits is knowable. Sizing needs pool wait and database metrics. |
| "This unused method would load everything if called" | It is never called. At most, note it as unreferenced. |
| "The ORM probably joins here" | The call is `CONFIRMED`, the SQL `INFERRED` for the version, the exact SQL `UNKNOWN`. Say how to capture it. |
| "The README says it's PostgreSQL" | Driver, schema, and infrastructure decide. Record the drift. |
| "While I'm here: auth, validation, error middleware" | Outside scope. One line each under Open Questions. |
| "More findings show thoroughness" | Impact over count. Merge related findings along the path; drop the weak ones. |
| "A chart or Mermaid diagram would be clearer" | Output contract: plain text in `.md`, HTML/CSS in `.html`. |

## Relationship to Other MDW Skills

```text
distributed-flow       Where does the system communicate?
sequence-flow          What executes, in what order?
database-performance   How does the application use the database, and where can that become a bottleneck?
```

They compose. For one path, sequence-flow shows `InvoiceHandler.List() ->
InvoiceService.ListForAccount() -> for each invoice: LineRepo.ByInvoice()`. database-performance
adds what the database is asked to do there: `SELECT ... FROM invoice_lines WHERE invoice_id =
$1` executed N times, no index on `invoice_id` after migration 009, statements `2 + N`, cost
growing with N times the size of `invoice_lines`.

Each skill works alone. When another skill's artifact exists for a path in scope (by default
under `mdw/distributed-flow/` or `mdw/sequence-flow/`), use it as the map of hops and verify what
you rely on. Do not reproduce its flow. Transaction correctness (atomicity, dual writes, outbox,
sagas) belongs to `distributed-flow`; this skill covers what transactions cost. SQL injection
and other security risks in query construction belong to the `security` skill; list them under
Outside scope.

## References

Read each reference when its step needs it. They extend this workflow; they do not repeat it.

| File | Read when |
|---|---|
| `references/query-analysis.md` | Steps 2 to 4: engine detection, effective schema, query discovery, resolving SQL, frequency and cost driver, inventory |
| `references/query-patterns.md` | Step 5, and whenever you are about to call something N+1 or a scan |
| `references/orm-performance.md` | Steps 4 and 5 for any ORM: how to see the SQL, loading behavior, hidden queries |
| `references/indexing.md` | Step 6: alignment rules, engine differences, candidates, redundancy |
| `references/transaction-performance.md` | Step 7 |
| `references/locking-concurrency.md` | Step 8 |
| `references/connection-pool.md` | Step 9 |
| `references/pagination.md` | Step 10 |
| `references/caching.md` | Step 11 |
| `references/database-evidence.md` | Whenever runtime evidence exists, and for every validation plan (steps 12 to 14) |
| `references/report-format.md` | Step 15 |
