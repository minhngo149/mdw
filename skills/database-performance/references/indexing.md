# Index Analysis

How to decide, from source code and the effective schema, whether a query's predicates, joins,
and ordering are served by an index, and how to report candidate, misaligned, and potentially
redundant indexes without claiming what the optimizer does.

Without an execution plan, every conclusion here about how the database executes a query is
`INFERRED`. Whether an index exists is `CONFIRMED` from the effective schema
(`query-analysis.md` section 2).

## Contents

1. B-tree rules that static analysis can apply
2. Checking a query against the indexes
3. Predicates that defeat an index
4. Engine differences
5. Missing index candidates
6. Redundant and overlapping indexes
7. Foreign keys, soft delete, partial and expression indexes
8. Trade-offs and rollout
9. Validation

## 1. B-tree rules that static analysis can apply

These hold for B-tree indexes in PostgreSQL, MySQL InnoDB, SQLite, and SQL Server rowstore
tables. Other index types (GIN, GiST, BRIN, hash, full-text) follow their own rules.

- **Leftmost prefix.** An index on `(a, b, c)` can seek on `a`, on `a, b`, or on `a, b, c`. It
  generally cannot seek on `b` or `c` alone. PostgreSQL 18+ and MySQL 8.0.13+ can *skip scan*
  when the leading column has few distinct values; treat that as possible, never as given.
- **Equality first, then one range.** Columns compared with `=` (or `IS NULL`) can be followed
  by one range column (`>`, `<`, `BETWEEN`, prefix `LIKE`). Columns after the first range
  column cannot narrow the seek; they can only filter entries inside the index.
- **Ordering.** An index can return rows already in `ORDER BY` order when the equality columns
  come first and the order columns follow in index order. A single direction is served by a
  forward or backward scan, so `(a, created_at)` serves `WHERE a = ? ORDER BY created_at DESC`;
  a `DESC` index is not needed for that. Mixed directions (`ORDER BY x ASC, y DESC`) need an
  index declared with matching or fully mirrored directions.
- **Order plus limit.** When the index provides the order, `LIMIT n` lets the database stop
  after n matching rows. Without it, the database reads every matching row and sorts (or runs a
  top-N sort) before applying the limit. This is the most common reason a `LIMIT 20` query still
  grows with the data.
- **A range breaks the order after it.** With `WHERE a = ? AND b > ? ORDER BY c`, the index
  `(a, b, c)` cannot provide `c` order; `(a, c)` can, and filters `b` as it goes.
- **IN and ANY.** An `IN` list or `= ANY(...)` on an index column is several seeks; it may stop
  the index from providing the `ORDER BY` order. Verify with a plan.
- **Covering.** If every column the query reads is in the index, the table may not be read at
  all (PostgreSQL *index-only scan*, which also depends on the visibility map; MySQL `Using
  index`). PostgreSQL 11+ and SQL Server add non-key columns with `INCLUDE`. `SELECT *` defeats
  covering.
- **InnoDB secondary indexes end with the primary key.** In MySQL, `(a)` behaves as `(a, id)`,
  so it serves `WHERE a = ? ORDER BY id`.
- **Selectivity.** An index on a column with few distinct values (booleans, a status with three
  values) rarely helps on its own unless the queried value is rare. The optimizer may correctly
  prefer a full scan.

## 2. Checking a query against the indexes

For each query in the inventory that is on a frequent path or has a growing cost driver:

1. Write down its access pattern: equality columns, range columns, join columns, `ORDER BY`
   columns and directions, `GROUP BY` columns, selected columns, and hidden predicates added by
   the ORM (soft delete, tenant scope).
2. List the effective indexes on the table, in column order, with type and predicate.
3. Decide the alignment for each candidate index, and name the rule from section 1 that decides
   it.

Alignment of one index, `(tenant_id, status, created_at)`, with several queries:

| Query shape | Alignment | Why |
|---|---|---|
| `WHERE tenant_id = ?` | aligned | leading column |
| `WHERE tenant_id = ? AND status = ?` | aligned | prefix |
| `WHERE tenant_id = ? AND status = ? ORDER BY created_at DESC LIMIT 20` | aligned | equality prefix, then the order column; backward scan; can stop at 20 |
| `WHERE tenant_id = ? ORDER BY created_at DESC LIMIT 20` | partial | `status` sits between; the index cannot give `created_at` order across statuses, so a sort of the tenant's rows is likely |
| `WHERE tenant_id = ? AND created_at > ?` | partial | seeks on `tenant_id`; `created_at` only filters inside the index |
| `WHERE status = ?` | not aligned | leading column missing; skip scan possible on PostgreSQL 18+ and MySQL 8.0.13+ |
| `ORDER BY created_at DESC LIMIT 50` (no filter) | not aligned | no index leads with `created_at` |

Write alignment as a claim with its label: "not aligned (`CONFIRMED` from the schema); the
database likely sorts all of the tenant's rows (`INFERRED`); execution plan not available".

## 3. Predicates that defeat an index

| Pattern | Why a plain B-tree index does not help | Options, each needing verification |
|---|---|---|
| Function or cast on the column: `lower(email) = ?`, `date(created_at) = ?`, `created_at::date = ?` | the index stores column values, not function results | expression index (PostgreSQL, SQLite; MySQL 8.0.13+ functional key parts; a generated column plus index in MySQL 5.7+); rewrite dates as a half-open range; a case-insensitive type or collation (PostgreSQL `citext` or a nondeterministic collation). MySQL's default `_ci` collations already compare case-insensitively, so `lower()` there is usually unnecessary and blocks the index |
| Leading wildcard: `LIKE '%x%'`, `ILIKE '%x%'` | B-trees are ordered by prefix | PostgreSQL `pg_trgm` GIN or GiST index, or full-text search (`tsvector`); MySQL `FULLTEXT` index with `MATCH ... AGAINST`; SQLite FTS5; an external search engine. All change matching semantics (tokens, stop words, minimum lengths); say so |
| Prefix `LIKE 'x%'` in PostgreSQL | uses a B-tree only with the `C` collation or a `text_pattern_ops`/`varchar_pattern_ops` index | a pattern-ops index |
| Type mismatch: string column compared with a number, UUID stored as text, parameter bound as a different type | the conversion is applied to the column | match the parameter type to the column |
| Collation or charset mismatch in a join (common in MySQL) | the comparison cannot use the index order | align column definitions |
| `OR` across different columns | one index cannot serve both branches | PostgreSQL BitmapOr or MySQL index merge may combine two indexes; a `UNION ALL` rewrite |
| Arithmetic on the column: `amount * 1.2 > ?` | same as a function | move the arithmetic to the parameter |
| JSON path: `data->>'k' = ?`, `JSON_EXTRACT(data, '$.k') = ?` | a column index does not index paths | PostgreSQL expression index on the path, or GIN on `jsonb` for containment (`@>`); MySQL generated column plus index, or a multi-valued index (8.0.17+) |
| Catch-all optional filter: `WHERE ($1 IS NULL OR col = $1)` | one plan must serve both cases; PostgreSQL generic plans for prepared statements make this worse | build the predicate only when the filter is present |
| `!=`, `NOT IN`, `NOT LIKE` | usually match most rows | often a scan is correct; not a finding by itself |

## 4. Engine differences

Do not transfer advice between engines. Check the engine and version first.

| Feature | PostgreSQL | MySQL InnoDB | SQLite | SQL Server |
|---|---|---|---|---|
| Index created for FK column automatically | no | yes, if no usable index exists | no | no |
| Partial or filtered index | yes (`WHERE`) | no | yes (3.8+) | yes (filtered) |
| Expression index | yes | 8.0.13+ functional key parts; generated columns earlier | yes (3.9+) | via computed columns |
| `INCLUDE` columns | 11+ | no (primary key appended implicitly) | no | yes |
| `DESC` key order honored | yes | 8.0+ (MariaDB 10.8+) | yes | yes |
| Build without blocking writes | `CREATE INDEX CONCURRENTLY` (not inside a transaction) | online DDL for most secondary indexes | no | `ONLINE = ON` (edition-dependent) |
| Text search index | GIN on `tsvector`, `pg_trgm` | `FULLTEXT` | FTS5 | Full-Text Search |
| Index usage statistics | `pg_stat_user_indexes.idx_scan` | `sys.schema_unused_indexes`, `performance_schema.table_io_waits_summary_by_index_usage` | none built in | `sys.dm_db_index_usage_stats` |

MariaDB and MySQL have diverged; check MariaDB features against its own version. MongoDB
compound indexes follow a similar prefix rule; order fields Equality, Sort, Range (the ESR
guideline), and verify with `explain("executionStats")`.

## 5. Missing index candidates

A candidate is a hypothesis to validate, never a migration to apply. Report one only when all of
these hold:

- the query is on a path whose frequency class is not `RARE`, or its cost driver is `TABLE`
  on a table with growth signals
- no effective index serves its predicate or ordering (checked with section 1, including
  backward scans, leftmost prefixes, and the InnoDB primary-key suffix)
- the predicate is sargable, or fixing the predicate is part of the candidate

Format:

```text
CANDIDATE: invoices (account_id, issued_at)
  Query       SELECT id, number, total, issued_at FROM invoices
              WHERE account_id = $1 ORDER BY issued_at DESC LIMIT 20
  Location    internal/billing/repo.go InvoiceRepo.ListByAccount
  Path        GET /accounts/{id}/invoices  (PER_REQUEST)
  Existing    invoices_account_id_idx (account_id)             migrations/004_invoices.sql
  Why         equality on account_id, then newest-first order. The existing index finds the
              account's rows but not in issued_at order, so each call likely reads all of the
              account's invoices and sorts them before LIMIT (INFERRED; plan not available)
  Trade-offs  maintained on every INSERT and on UPDATEs of account_id or issued_at; storage;
              invoices_account_id_idx becomes a prefix of it (POTENTIALLY REDUNDANT)
  Validate    EXPLAIN (ANALYZE, BUFFERS) before and after, on production-like data, for an
              account with many invoices: Sort node present or absent, rows read, buffers
  Rollout     not generated. If adopted: CREATE INDEX CONCURRENTLY, outside a transaction
```

Do not recommend an index when the table is a small lookup table that stays small, the query is
rare while the table is written on a hot path, the column has low selectivity for the values
queried, an existing index already serves the query, or the predicate itself is non-sargable
(fix the predicate first).

## 6. Redundant and overlapping indexes

Label these `POTENTIALLY REDUNDANT`. Never write "remove this index"; removal needs workload
evidence the repository does not contain.

| Case | Assessment |
|---|---|
| Exact duplicate (same columns, order, type, predicate) | `POTENTIALLY REDUNDANT`; check whether one backs a constraint |
| Prefix of another, `(a)` and `(a, b)` | `POTENTIALLY REDUNDANT`; `(a, b)` can serve most lookups on `a`, but the narrower index is smaller and may still be preferred for some queries |
| A unique index that is a prefix of another | not redundant: it enforces a constraint |
| An index backing a primary key, unique constraint, or (MySQL) foreign key | not removable without changing the constraint |
| Different type, predicate, expression, or collation | not redundant |
| `(a, b)` and `(b, a)` | not redundant: they serve different leading-column queries |

Evidence that would support removal: index usage statistics covering a full business cycle
(month-end jobs, seasonal traffic), from **every** server that runs queries. Statistics are per
server, so an index used only on a read replica shows zero scans on the primary. Also check
statistics resets.

## 7. Foreign keys, soft delete, partial and expression indexes

**Foreign keys.** In PostgreSQL, SQLite, and SQL Server, a foreign key does not create an index
on the referencing column. An unindexed referencing column affects:

- joins and lookups from parent to children (`WHERE child.parent_id = ?`)
- every `DELETE` of a parent row, and every `UPDATE` of its key, because the database must look
  for referencing rows (for `RESTRICT`, `NO ACTION`, `CASCADE`, and `SET NULL` alike)

In MySQL InnoDB, a usable index exists by construction (`INFERRED` from the engine; verify with
`SHOW INDEX`). Frameworks may add FK indexes themselves: see `query-analysis.md` section 2.

**Soft delete.** Queries with `deleted_at IS NULL` (often added invisibly by GORM `DeletedAt`,
Rails `paranoia`/`discard`, Laravel `SoftDeletes`, Hibernate `@SQLRestriction`/`@Where`) are
filtered after the index lookup unless an index includes the predicate. Whether that matters
depends on the fraction of deleted rows, which is `UNKNOWN`. A PostgreSQL, SQLite, or SQL Server
partial index `WHERE deleted_at IS NULL` helps only if every relevant query includes exactly that
predicate and deleted rows are a meaningful fraction. Report the dependency; do not recommend a
partial index by default.

**Partial indexes** for rare values ("pending" jobs in a mostly "done" table) are often the
right shape for queue tables in PostgreSQL. The query predicate must imply the index predicate.

**Expression indexes** must match the expression exactly as written in the query, including
casts and function arguments.

## 8. Trade-offs and rollout

State the costs of every index recommendation:

- **Write cost.** Every `INSERT` writes every index; every `UPDATE` of an indexed column writes
  that index. In PostgreSQL, updating any indexed column also prevents HOT updates, which
  increases bloat.
- **Storage and memory.** More indexes compete for the buffer cache.
- **Build locking.** A plain `CREATE INDEX` in PostgreSQL blocks writes to the table for the
  whole build. `CONCURRENTLY` avoids that but takes longer, cannot run inside a transaction
  (migration tools need their per-migration no-transaction option: Rails
  `disable_ddl_transaction!`, Django `atomic = False` with `AddIndexConcurrently`, Flyway
  `executeInTransaction=false`, golang-migrate one statement per file), and can leave an
  `INVALID` index if it fails. MySQL online DDL still takes brief metadata locks that queue
  behind long transactions.
- **Plan changes.** A new index can change plans for other queries on the same table.

## 9. Validation

- Compare `EXPLAIN` output before and after on production-like data. On small development data
  the optimizer correctly prefers full scans, so a missing index looks harmless there.
- PostgreSQL: `hypopg` creates hypothetical indexes for `EXPLAIN` without building them.
  `SET enable_seqscan = off` shows only whether an index *can* be used, not whether it is
  better.
- After deployment: index usage statistics (section 4 table) and the query's own statistics
  (`pg_stat_statements`, `performance_schema` digests). See `database-evidence.md`.
