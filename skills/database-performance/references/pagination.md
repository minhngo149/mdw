# Pagination and Batch Operations

How to recognize the pagination style, decide whether OFFSET depth or totals can become
expensive, and review batch reads and writes in jobs, imports, and exports. Neither offset nor
keyset pagination is better in general; the analysis states which trade-off this code has made
and whether its bounds hold.

## Contents

1. Recognize the style
2. OFFSET: cost and when it matters
3. Keyset (cursor) pagination: requirements
4. Totals and page counts
5. Ordering stability
6. Iterating a whole table in jobs and exports
7. Batch writes
8. Validation

## 1. Recognize the style

| Style | Evidence |
|---|---|
| Offset | `LIMIT ? OFFSET ?`, `OFFSET ... FETCH NEXT`, GORM `.Offset().Limit()`, Prisma `skip`/`take`, Django slicing `qs[a:b]` and `Paginator`, DRF `PageNumberPagination`/`LimitOffsetPagination`, Rails `.offset`, kaminari, will_paginate, Spring Data `Pageable`, Laravel `paginate()`/`simplePaginate()`, Sequelize `offset`, Mongo `skip()` |
| Keyset | `WHERE (sort_col, id) < ($1, $2) ORDER BY sort_col DESC, id DESC LIMIT n`, `WHERE id > $last ORDER BY id`, DRF `CursorPagination`, Laravel `cursorPaginate()`, Rails `find_each`, Prisma `cursor` (verify its SQL), GraphQL connection cursors that encode sort values |
| Opaque "cursor" | a cursor parameter that decodes to an offset or page number is still offset pagination. Read the decoder |
| None | a list endpoint with no limit (an unbounded read; see `query-patterns.md` section 4) |

Record where `page`, `offset`, `limit`, and `page_size` come from, whether each is validated,
and the maximum for each (a clamp in the handler, a validation rule, or none).

## 2. OFFSET: cost and when it matters

To return rows at `OFFSET k`, the database produces and discards the first k rows in order.
Even with an index that matches the filter and order, work grows with `k + limit`; without one,
every matching row is read and sorted first. The cost driver is `OFFSET`.

It matters when:

- the page or offset is user-controlled with no maximum (deep pages from crawlers, scripts, or
  a "last page" link)
- the filtered set per request can be large (a whole table, a large tenant)
- the endpoint is used to iterate everything (sync clients, exports, admin tools)
- each page also runs a `COUNT` of the full set (section 4)

It is usually fine when the maximum depth is bounded and small, or the set per key is small.

Offset also has a correctness problem under concurrent writes: inserts and deletes between page
requests shift rows, so clients see duplicates or miss rows.

## 3. Keyset (cursor) pagination: requirements

| Requirement | Why |
|---|---|
| A total order: the sort columns plus a unique tie-breaker (usually the primary key) | otherwise rows with equal sort values repeat or vanish across pages |
| An index on (equality filter columns, sort columns, tie-breaker) in matching direction | otherwise the database still sorts every matching row |
| A seek predicate matching the order | PostgreSQL supports row comparison `(a, b) < ($1, $2)` with index use. In MySQL, check the plan; the expanded form `a < ? OR (a = ? AND b < ?)` is the portable alternative. Mixed sort directions need the expanded form |
| Handling for NULLs in sort columns | NULL ordering breaks naive comparisons |
| A cursor encoding the last row's sort values | not an offset |

Trade-offs to state with any recommendation:

| | Offset | Keyset |
|---|---|---|
| Deep pages | cost grows with depth | cost independent of depth (with the index) |
| Jump to page N, "page 7 of 40" | yes | no; only next and previous |
| Concurrent inserts and deletes | rows shift between pages | stable |
| Changing the sort field | any column | each supported sort needs its own index and cursor format |
| Implementation | simple | more complex: cursor encoding, two directions, NULLs |

## 4. Totals and page counts

A page that shows a total runs a `COUNT` over every matching row on each request. Frameworks add
it implicitly: Spring Data `Page<T>` runs a count query and `Slice<T>` does not; Laravel
`paginate()` counts and `simplePaginate()` does not; DRF `PageNumberPagination` counts; Sequelize
`findAndCountAll` counts, and with `include` may need `distinct: true` to count correctly.

The count's cost driver is the whole matching set (`KEY_FANOUT` or `TABLE`), whatever the page
size. Options, each changing the UI: "has more" with `LIMIT n + 1`; an estimated count (the
PostgreSQL planner estimate, `pg_class.reltuples`, MySQL `information_schema.TABLES.TABLE_ROWS`,
both approximate); a capped count; a cached count. When the UI needs the exact number, keep the
count and report its cost driver as conditional.

## 5. Ordering stability

`ORDER BY created_at LIMIT 20 OFFSET 40` with duplicate `created_at` values gives an undefined
order among ties. PostgreSQL and MySQL may return ties in a different order on each execution,
so pages overlap or skip rows. Report it as a correctness finding and suggest a unique
tie-breaker. The index that supports the order should include the tie-breaker.

## 6. Iterating a whole table in jobs and exports

| Pattern | Total work for a table of n rows in pages of B |
|---|---|
| `LIMIT B OFFSET k*B` in a loop | about n^2 / (2B) rows produced and discarded over the run: grows quadratically (`INFERRED` from the offset mechanics) |
| Keyset by primary key: `WHERE id > $last ORDER BY id LIMIT B` | about n rows |
| Load everything at once (`SELECT id FROM t` into memory) | one query, but memory grows with n |
| Server-side cursor or streaming (PostgreSQL `DECLARE ... CURSOR`, Django `.iterator()`, JDBC fetch size, pgx row streaming) | about n rows, but the transaction or connection stays open for the whole run |

ORM batch iterators: Rails `find_each`/`in_batches` (keyset by primary key); Laravel
`chunkById`/`lazyById` (keyset) versus `chunk` (offset, and it skips rows when the loop modifies
the filtered column); Django `.iterator(chunk_size=...)`; Doctrine `toIterable()` with periodic
`clear()`; Spring Batch readers (check paging vs cursor).

Also record for each job: the schedule, the number of instances that run it (a job started in
every API replica runs once per replica unless something elects a leader), overlap protection,
and whether the per-row work inside the iteration issues more queries (`query-patterns.md`
section 3).

## 7. Batch writes

Recommend only what the engine, driver, and repository support.

| Engine or library | Bulk write mechanisms |
|---|---|
| PostgreSQL | multi-row `INSERT ... VALUES`; `INSERT ... SELECT FROM unnest($1::bigint[], ...)`; `COPY FROM STDIN` (pgx `CopyFrom`, psycopg `copy`, `pg-copy-streams`); `INSERT ... ON CONFLICT` for upserts; pgx `SendBatch` pipelines N statements in one round trip; JDBC `reWriteBatchedInserts=true` |
| MySQL | multi-row `INSERT`; `INSERT ... ON DUPLICATE KEY UPDATE`; `LOAD DATA [LOCAL] INFILE` (needs server and client settings); Connector/J `rewriteBatchedStatements=true` |
| GORM, Ent, sqlc, sqlx | `CreateInBatches`, `Create(&slice)`; `CreateBulk`; `:copyfrom` and `:batchexec`; `NamedExec` with a slice |
| Prisma | `createMany`; `updateMany`/`deleteMany` (same values for all rows) |
| Hibernate / JPA | `hibernate.jdbc.batch_size` with `order_inserts`/`order_updates`; `IDENTITY` id generation disables insert batching (a sequence with a pooled optimizer does not) |
| Django | `bulk_create(batch_size=...)`, `bulk_update`, `QuerySet.update()`, `F()` expressions |
| ActiveRecord | `insert_all`/`upsert_all`, `update_all`, `delete_all` (versus `destroy_all`, which loads and destroys one by one with callbacks) |
| Sequelize, TypeORM, Laravel, EF Core | `bulkCreate`; `insert().values([...])`; `insert([...])`/`upsert`; `SaveChanges` batching, `ExecuteUpdate`/`ExecuteDelete` (EF Core 7+) |

Batch size trade-offs: parameter and packet limits (`query-patterns.md` section 10); lock
footprint and duration; a burst of WAL or binlog that can lag replicas; memory; all-or-nothing
failure; skipped ORM hooks and validations; the order of returned IDs. A batch size in a
recommendation is a starting point to validate, not a measured optimum.

## 8. Validation

- Deep offsets: run the query with `EXPLAIN (ANALYZE, BUFFERS)` (PostgreSQL) or
  `EXPLAIN ANALYZE` (MySQL 8.0.18+) at offset 0 and at a depth taken from real access logs, on
  production-like data. Compare rows read and time.
- Totals: time the count query separately from the page query.
- Jobs: statements per run, run duration against table size, rows per second. For offset
  iteration, duration should grow faster than linearly as the table grows.
- Batch writes: statements, round trips, and commits per logical operation before and after;
  replica lag during the batch.
