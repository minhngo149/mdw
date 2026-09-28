# ORM Performance

ORMs decide the SQL that runs, often far from the call site: lazy loads in serializers,
default scopes, hooks, auto-saved associations. This reference lists, per ORM, how to see the
SQL, how relations load, and the behaviors that multiply or hide queries.

## Contents

1. Rules for ORM claims
2. Seeing the SQL
3. Go: GORM, Ent, sqlc, sqlx, pgx
4. Node: Prisma, TypeORM, Sequelize, query builders
5. JVM: Hibernate, JPA, Spring Data
6. Python: Django ORM, SQLAlchemy
7. Ruby: ActiveRecord
8. PHP: Eloquent, Doctrine
9. .NET: EF Core
10. MongoDB: drivers and Mongoose
11. GraphQL resolvers

## 1. Rules for ORM claims

- Record the ORM and its **resolved** version from the lockfile (`go.sum`, `package-lock.json`,
  `pnpm-lock.yaml`, `yarn.lock`, `poetry.lock`, `Gemfile.lock`, `composer.lock`,
  `gradle.lockfile`, or the resolved `pom.xml` version). Behavior below changes between
  versions.
- The ORM call and its arguments are `CONFIRMED`. The SQL it generates is `INFERRED` from
  documented behavior for that version. The exact SQL, and the number of statements, are
  `UNKNOWN` until captured from a log (`RUNTIME`).

```text
CONFIRMED  InvoiceService.List calls db.Preload("Lines").Find(&invoices)
INFERRED   GORM v2 loads Lines with a second query, WHERE invoice_id IN (<ids of the page>)
UNKNOWN    exact SQL and plan under the current configuration (resolve by enabling the GORM logger)
```

- Configuration changes behavior: global loading defaults, default scopes, soft-delete
  plugins, hooks and callbacks, naming strategies, and auto-migration. Read them before
  applying the tables below.
- When the repository has a test that asserts query counts, it is strong evidence of intended
  behavior. It is not `RUNTIME` evidence of production behavior.

## 2. Seeing the SQL

| ORM | Log SQL | Count queries in tests |
|---|---|---|
| GORM | `db.Debug()`, `logger.Default.LogMode(logger.Info)` | a logger or callback that counts |
| Ent | `client.Debug()` | a driver wrapper |
| `database/sql`, sqlx, sqlc, pgx | driver tracing: pgx `QueryTracer`, `otelsql`, `sqlhooks` | the same tracer |
| Prisma | `new PrismaClient({ log: ["query"] })`, `$on("query", ...)` | count `query` events |
| TypeORM / Sequelize | `logging: ["query"]` / `logging: console.log` | a logger that counts |
| Hibernate | `org.hibernate.SQL=DEBUG`, `hibernate.generate_statistics=true` | `Statistics.getPrepareStatementCount()`, datasource-proxy |
| Django | `django.db.backends` logger at DEBUG, `connection.queries` | `assertNumQueries`, `CaptureQueriesContext` |
| SQLAlchemy | `create_engine(..., echo=True)` | `before_cursor_execute` event listener |
| ActiveRecord | development log, `ActiveSupport::Notifications` `sql.active_record` | a notifications subscriber; the Bullet gem for N+1 |
| Eloquent | `DB::listen`, `DB::enableQueryLog()`, Debugbar, Telescope | `DB::getQueryLog()` |
| Doctrine | DBAL logging middleware, Symfony profiler | the profiler's query count |
| EF Core | `optionsBuilder.LogTo(...)` | a command interceptor |
| Mongoose | `mongoose.set("debug", true)` | the debug hook |

## 3. Go

**GORM (v2, `gorm.io/gorm`)**

- No lazy loading. Unloaded associations are zero values, so GORM N+1 only appears as explicit
  queries in loops.
- `Preload("Assoc")`: one extra query per preloaded association, using `IN` on the parent keys.
  `Joins("Assoc")`: a join, for has-one and belongs-to relations.
- `Create`/`Save` also upsert associations present on the struct (auto-save), which adds
  statements; `Omit(clause.Associations)` prevents it. `Save` writes every column.
- Each `Create`/`Update`/`Delete` runs in its own transaction unless `SkipDefaultTransaction:
  true`.
- `gorm.DeletedAt` adds `deleted_at IS NULL` to every query invisibly; `Unscoped()` removes it.
- `First`/`Last` add `ORDER BY` primary key and `LIMIT 1`; `Take` adds only the limit.
  `Find(&slice)` without `Limit` is unbounded.
- Bulk: `Create(&slice)` (one multi-row insert), `CreateInBatches(&slice, n)`.
- Hooks (`BeforeCreate`, `AfterFind`, ...) can issue queries per row.
- The pool is the underlying `database/sql` pool: `db.DB()` returns it.

**Ent**

- No lazy loading. Traversing an edge (`inv.QueryLines()`) on each entity in a loop is N+1.
- `With<Edge>()` eager loads with one extra query per edge per level.
- `CreateBulk` for bulk inserts. Indexes are declared in the schema's `Indexes()` and applied
  by migration; check the generated migrations or the auto-migration call.

**sqlc**: the SQL is in the query files (`CONFIRMED`). `:many` loads every row into a slice.
Bulk: `:copyfrom`, and `:batchexec`/`:batchmany`/`:batchone` with pgx. For lists, PostgreSQL uses
`= ANY(@ids::bigint[])`; MySQL and SQLite use `sqlc.slice()`.

**sqlx**: the SQL is literal. `Select` loads every row; `In` expands slices; `NamedExec` with a
slice of structs performs a multi-row insert.

**pgx native**: `Batch` + `SendBatch` pipelines statements; `CopyFrom`; prepared statements are
cached by default (see `connection-pool.md` section 7 for PgBouncer).

## 4. Node

**Prisma**

- Relations load only through `include`/`select`; there is no implicit lazy loading. The fluent
  API (`prisma.account.findUnique(...).invoices()`) runs a separate query.
- How `include` becomes SQL depends on the version and `relationLoadStrategy`: `query` issues
  one query per relation level; `join` (behind the `relationJoins` preview feature in Prisma 5)
  uses one query with JSON aggregation. Check the generator's `previewFeatures`, any
  `relationLoadStrategy` argument, and the version, and treat the shape as `INFERRED`.
- The dataloader: `findUnique` and `findUniqueOrThrow` calls issued **in the same tick** with the
  same selection are batched into one query. A sequential `await` in a loop is not batched.
  `findFirst`, `findMany`, `count`, and `aggregate` are never batched this way.
- `findMany` without `take` is unbounded. `skip`/`take` is offset pagination; `cursor` pagination
  generates a seek subquery (verify the SQL).
- `$transaction([...])` runs the listed operations in one transaction. The interactive
  `$transaction(async (tx) => ...)` holds a connection for the whole callback, with defaults
  `maxWait` 2 s and `timeout` 5 s (after which it is rolled back).
- Nested writes (`create` with nested `create`) run several statements in a transaction.
- Bulk: `createMany`; `updateMany`/`deleteMany` apply the same change to every matching row.
- Indexes: `@@index`, `@unique`, `@@unique`. With `relationMode = "prisma"`, no foreign keys are
  created, so on MySQL no index is created for relation columns either; Prisma warns that
  `@@index` is needed.
- Pool: `connection_limit` and `pool_timeout` URL parameters (`connection-pool.md`).

**TypeORM**

- `eager: true` on a relation loads it on every `find`, which is invisible at the call site.
  `lazy: true` relations are Promises that query on access, so awaiting them in a loop is N+1.
- `relations: [...]` and `leftJoinAndSelect` join; joining two to-many relations multiplies rows.
  `skip`/`take` with joins runs a separate query for the distinct IDs.
- `save()` selects the entity first to decide between insert and update; `insert()`/`update()`
  do not.

**Sequelize**

- `include` joins (`LEFT OUTER JOIN`); `separate: true` on a has-many include issues a separate
  query instead. Getter methods (`post.getComments()`) in a loop are N+1.
- `findAndCountAll` runs a count too; with includes, it may need `distinct: true` to count
  correctly. `raw: true` skips model instances. Bulk: `bulkCreate`.

**Knex, Kysely, Drizzle**: query builders. The clauses are visible, so the SQL shape is
`INFERRED` with high confidence; capture the text before reasoning about plans.

## 5. JVM: Hibernate, JPA, Spring Data

- JPA defaults: `@ManyToOne` and `@OneToOne` are `EAGER`; `@OneToMany` and `@ManyToMany` are
  `LAZY`. An `EAGER` to-one on a JPQL list query is loaded with a secondary select per distinct
  parent (N+1) unless the query fetch-joins it.
- Lazy collections touched in a loop or during serialization are N+1. Fixes: `JOIN FETCH`,
  `@EntityGraph`, `@BatchSize` or `hibernate.default_batch_fetch_size` (batch loads with `IN`),
  `@Fetch(FetchMode.SUBSELECT)`.
- A collection fetch join plus pagination makes Hibernate load every row and paginate in memory,
  logging a warning (`HHH000104` in Hibernate 5; `HHH90003004` in 6).
- Fetch-joining two `List` collections fails (`MultipleBagFetchException`); with `Set`s it
  produces a Cartesian product.
- Spring Boot `spring.jpa.open-in-view` defaults to `true` (`transaction-performance.md` section
  4).
- Spring Data: derived queries (`findByStatus`) are JPQL. `findAll()` is unbounded. `Page<T>`
  runs a count query; `Slice<T>` does not. `@Query(countQuery = ...)` controls it.
- Writes: `saveAll` sends one statement per entity unless JDBC batching is configured
  (`hibernate.jdbc.batch_size`, `order_inserts`, `order_updates`). `GenerationType.IDENTITY`
  disables insert batching.
- `@Transactional(readOnly = true)` lets Hibernate skip dirty checking. Large persistence
  contexts in batch jobs need periodic `flush()` and `clear()`.
- `@Modifying @Query("update ...")` runs a bulk statement and bypasses the persistence context.

## 6. Python

**Django ORM**

- QuerySets are lazy and evaluate on iteration, `len()`, `list()`, `bool()`, and slicing with a
  step.
- A forward foreign key accessed per object (`inv.account.name`) is a query per object unless
  `select_related("account")` (a join). A reverse foreign key or many-to-many
  (`inv.lines.all()`) is a query per object unless `prefetch_related("lines")` (one `IN` query).
  Calling `.filter()` on a prefetched relation issues a new query; use `Prefetch(queryset=...)`.
- `.only()`/`.defer()`: accessing a deferred field loads it per object.
- `if qs:` and `len(qs)` load every row; `.exists()` and `.count()` do not.
- `.iterator(chunk_size=...)` streams; `values()`/`values_list()` skip model instances.
- Bulk: `bulk_create`, `bulk_update`, `QuerySet.update()`, `F()` expressions. They skip
  `save()` and signals.
- `ForeignKey` creates an index by default (`db_index=True`).
- DRF: nested serializers and `SerializerMethodField` cause N+1 unless the view's queryset
  selects or prefetches; `PageNumberPagination` runs a count. Admin `list_display` over a foreign
  key needs `list_select_related`.

**SQLAlchemy**

- `relationship()` defaults to `lazy="select"`: a query on first access per object. Options:
  `joinedload`, `selectinload` (`IN`), `subqueryload`, and `raiseload` to forbid lazy loads.
- With the asyncio extension, lazy loads raise instead of querying, which forces explicit
  loading.
- `expire_on_commit=True` (the default) expires every loaded object at commit, so reading an
  attribute afterwards reloads it: a query per object after each commit.
- `yield_per` streams. `insert().values([...])` and 2.0's batched `executemany` handle bulk
  inserts. Autoflush runs before queries.

## 7. Ruby: ActiveRecord

- Associations are lazy. `includes` (chooses between `preload` and `eager_load`), `preload`
  (`IN` query), `eager_load` (`LEFT OUTER JOIN`). `strict_loading` (Rails 6.1+) raises on lazy
  loads; the Bullet gem detects N+1 in development.
- `present?`/`blank?` on a relation load every record. `exists?`/`any?` on an unloaded relation
  issue `SELECT 1 ... LIMIT 1`. `count` always queries; `size` counts or uses the loaded records;
  `length` loads.
- `find_each`/`in_batches` iterate by primary key (1000 per batch by default). `pluck` skips
  model instances.
- `insert_all`/`upsert_all` skip validations and callbacks. `destroy_all` and
  `dependent: :destroy` load and destroy each row with callbacks (N statements, cascading);
  `delete_all` and `dependent: :delete_all` issue one statement.
- `counter_cache` and `touch: true` update the parent row on every child write (a hot row).
- `validates :x, uniqueness: true` runs a `SELECT` on every save, and races without a unique
  index.
- `t.references`/`add_reference` add an index by default. `default_scope` (and soft-delete gems)
  add hidden predicates. `after_save` runs inside the transaction; `after_commit` after it.

## 8. PHP: Eloquent, Doctrine

**Eloquent**

- Relations are lazy. `with()` eager loads with one `IN` query per relation; `withCount()`
  counts in SQL. `Model::preventLazyLoading()` turns lazy loads into errors.
- `$with` on a model eager loads on every query; accessors and `$appends` can query on every
  serialization.
- `chunk()` pages with `OFFSET`; `chunkById()`/`lazyById()` use keyset. `cursor()` hydrates one
  model at a time, but PDO with MySQL buffers the full result by default.
- `paginate()` counts; `simplePaginate()` does not; `cursorPaginate()` is keyset.

**Doctrine**

- Associations are lazy (proxies) by default; `fetch="EAGER"` loads them always. A DQL fetch join
  (`JOIN ... SELECT`) loads them in one query. `EXTRA_LAZY` collections answer `count()` and
  `contains()` without loading.
- The `Paginator` with fetch-joined collections issues several queries. Batch processing needs
  periodic `flush()` and `clear()`. Array hydration is cheaper than objects.

## 9. .NET: EF Core

- Lazy loading is off unless proxies or `ILazyLoader` are configured; when on, navigation access
  in loops is N+1.
- `Include`/`ThenInclude` join in one query; several collection includes produce a Cartesian
  product, which `AsSplitQuery()` (EF Core 5+) avoids with one query per collection.
- `AsNoTracking()` for read-only queries. `ExecuteUpdate`/`ExecuteDelete` (EF Core 7+) run
  set-based statements. `SaveChanges` batches its statements in one transaction.

## 10. MongoDB: drivers and Mongoose

- `populate()` issues one extra query per path with `$in` (not N+1); calling `populate()` or
  `findById` per document in a loop is N+1.
- `lean()` skips document hydration. `skip()` has the same depth cost as `OFFSET`.
  `countDocuments` scans matches; `estimatedDocumentCount` uses metadata.
- Indexes come from `schema.index()` and `createIndex` calls. Mongoose `autoIndex` (on by
  default) builds indexes at application start.
- Unbounded arrays inside documents grow the document (the document size limit is 16 MB) and
  every read of it.
- Verify with `explain("executionStats")`: `COLLSCAN` vs `IXSCAN`, and `totalDocsExamined`
  against `nReturned`.

## 11. GraphQL resolvers

Every field resolver that queries runs once per parent object. A list of N parents with a
resolved relation is `1 + N` unless a request-scoped loader (DataLoader, gqlgen dataloaden,
graphql-batch, Strawberry/Graphene DataLoader) batches it. Check that the loader is created per
request, not globally (a global loader is a cache with no invalidation). Nested lists multiply:
`1 + N + N x M`.
