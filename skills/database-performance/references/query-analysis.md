# Query Analysis

How to find the database, reconstruct the schema the code actually runs against, find every
query that matters, and connect each one to the code path that executes it. The output of this
reference is the **query inventory** that every later step reads.

## Contents

1. Database discovery
2. Schema discovery: the effective schema
3. Query discovery
4. Resolving the SQL that actually runs
5. Connecting a query to its execution path
6. Frequency and cost driver
7. The query inventory
8. Large repositories: choosing what to analyze

## 1. Database discovery

Identify each database the application talks to. Every signal below is a lead until two agree;
in-repo infrastructure and driver evidence outrank documentation.

| Signal | Where to look | Examples |
|---|---|---|
| Driver dependency | `go.mod`, `package.json`, `pyproject.toml`/`requirements*.txt`, `pom.xml`/`build.gradle`, `Gemfile`, `composer.json`, `*.csproj` | PostgreSQL: `jackc/pgx`, `lib/pq`, `pg`, `postgres`, `psycopg`/`psycopg2`, `asyncpg`, `org.postgresql:postgresql`, `Npgsql`. MySQL/MariaDB: `go-sql-driver/mysql`, `mysql2`, `PyMySQL`, `mysqlclient`, `mysql-connector-j`, `mariadb-java-client`, `MySqlConnector`. SQLite: `mattn/go-sqlite3`, `modernc.org/sqlite`, `better-sqlite3`, `sqlite-jdbc`. SQL Server: `go-mssqldb`, `tedious`/`mssql`, `mssql-jdbc`, `Microsoft.Data.SqlClient`. MongoDB: `mongo-driver`, `mongodb`/`mongoose`, `pymongo`/`motor` |
| ORM or tool config | schema and config files | Prisma `datasource db { provider = "mysql" }`, Django `DATABASES.ENGINE`, Rails `database.yml adapter:`, Hibernate dialect or JDBC URL, TypeORM `type:`, Sequelize `dialect:`, Knex `client:`, GORM `gorm.io/driver/postgres`, Ent `dialect.Postgres`, sqlc `engine:`, Laravel `DB_CONNECTION` |
| Connection string | `.env.example`, config files, compose, manifests | `postgres://`, `mysql://`, `jdbc:postgresql:`, `sqlserver://`, `mongodb+srv://`, `file:app.db` |
| Infrastructure | compose, Helm, Kubernetes, Terraform, CI service containers | `image: postgres:16`, `image: mysql:8.0`, `aws_db_instance.engine`, `aws_rds_cluster.engine = "aurora-postgresql"` |
| SQL dialect | queries and migrations | PostgreSQL: `$1`, `RETURNING`, `JSONB`, `ILIKE`, `::type`, `ON CONFLICT`. MySQL: backticks, `AUTO_INCREMENT`, `ENGINE=InnoDB`, `ON DUPLICATE KEY`. SQL Server: `TOP`, `NVARCHAR`, `IDENTITY`. SQLite: `AUTOINCREMENT`, `PRAGMA` |
| README, ADRs, comments | docs | leads only. When they disagree with the evidence above, record documentation drift |

**Version.** Take it from an image tag, `engine_version`, or a CI service container. `latest`,
no tag, or a local-only compose file makes the production version `UNKNOWN`; say which features
you rely on that depend on the version (for example PostgreSQL 11+ `INCLUDE`, MySQL 8.0.13+
functional index parts).

**Wire-compatible engines.** A PostgreSQL driver proves only the protocol. CockroachDB,
YugabyteDB, Aurora PostgreSQL, AlloyDB, and Neon speak it with different planners, locking, and
index features; PlanetScale/Vitess, TiDB, and Aurora MySQL do the same for MySQL. Name the
engine from infrastructure evidence; with only a driver, write "PostgreSQL protocol; engine
INFERRED as PostgreSQL" and keep engine-specific advice conditional.

**Several databases.** Record each one and which deployable and repository layer uses it.
Every query in the inventory names its database.

Also record: the driver and its version, the ORM or query builder and its version (from the
lockfile, not the manifest range), the migration tool, where the pool is constructed, poolers
or proxies (PgBouncer, RDS Proxy, ProxySQL), and read replicas or read/write splitting.

## 2. Schema discovery: the effective schema

The effective schema is what exists after **every** migration has run, in order. The first
migration is not the schema.

| Tool | Where migrations live |
|---|---|
| golang-migrate, goose, dbmate | `migrations/*.up.sql`, `-- +goose Up`, `-- migrate:up` |
| Flyway, Liquibase | `V<n>__*.sql`, `R__*.sql` (repeatable, re-applied when changed), changelog XML/YAML |
| Alembic | `versions/*.py` (`op.create_index`, `op.drop_index`, `op.alter_column`) |
| Django | `<app>/migrations/*.py` (`AddIndex`, `RemoveIndex`, `AlterField`); models `db_index=True`, `Meta.indexes` |
| Rails | `db/migrate/*.rb`; `db/schema.rb` or `db/structure.sql` is the dumped current state |
| Prisma | `prisma/migrations/*/migration.sql` plus `schema.prisma` |
| Knex, TypeORM, Sequelize, Laravel, EF Core, Atlas, Ent | their migrations directory; EF Core `ModelSnapshot` is the dumped state |
| sqlc | the `schema:` path in `sqlc.yaml` |

Rules:

- Replay migrations in order. Track `CREATE INDEX`, `DROP INDEX`, `ALTER TABLE ... DROP
  CONSTRAINT`, renames, and column type changes. An index created in migration 1 and dropped in
  migration 5 does not exist.
- A dumped schema (`schema.rb`, `structure.sql`, a `pg_dump -s` file, `ModelSnapshot`) is strong
  evidence if it is at least as new as the latest migration. Compare its version marker with
  the migration list.
- ORM model declarations (struct tags, annotations, `@@index`) state intent. They are effective
  only if a migration contains them or the application auto-migrates at startup (GORM
  `AutoMigrate`, Hibernate `hbm2ddl.auto=update`, TypeORM `synchronize: true`, Ent
  `client.Schema.Create`). Find that call before treating models as the schema.
- Indexes created by hand in production, by a DBA, or by another repository are invisible
  here. Every "missing index" claim is about the repository's schema; the production schema is
  `UNKNOWN` unless the user supplies it (`\d+ table`, `SHOW INDEX FROM table`), in which case it
  is `RUNTIME` evidence and outranks the migrations.
- Framework defaults can create indexes the migrations do not spell out: Django creates an
  index for every `ForeignKey` unless `db_index=False`; Rails `t.references`/`add_reference`
  adds an index by default; MySQL InnoDB creates an index for a foreign key when no usable one
  exists. Record these as `INFERRED` with the reason.

Per table, record only what the analysis needs:

```text
TABLE invoices                                  migrations/004_invoices.sql
  pk          (id)
  fk          account_id -> accounts.id         indexed: invoices_account_id_idx (004)
  large       pdf BYTEA, line_snapshot JSONB
  soft delete deleted_at
  index       invoices_account_id_idx (account_id)                          004
  index       invoices_account_issued_idx (account_id, issued_at)           007
  dropped     invoices_status_idx                                           created 004, dropped 009
  triggers    invoices_audit AFTER UPDATE -> INSERT INTO audit_log          011
```

Also note partitioning, views and materialized views (and what refreshes them), generated
columns, check constraints that encode states, and triggers (hidden writes on every statement).

## 3. Query discovery

Search for call sites, not only SQL keywords. Use Grep or `rg -n`, and exclude vendored and
generated directories.

| Stack | Search for |
|---|---|
| Go `database/sql`, sqlx, pgx | `\.(Query|QueryRow|Exec)(Context)?\(`, `\.(Select|Get|NamedExec|NamedQuery)\(`, `SendBatch`, `CopyFrom` |
| GORM | `\.(Find|First|Take|Last|Where|Preload|Joins|Create|CreateInBatches|Save|Update|Updates|Delete|Raw|Exec|Count|Pluck|Scan)\(` on a `*gorm.DB` |
| Ent | `client\.\w+\.(Query|Create|CreateBulk|Update|Delete)\(`, `\.With\w+\(`, `\.Query\w+\(` |
| sqlc | `-- name: \w+ :(one|many|exec|execrows|batchexec|batchmany|batchone|copyfrom)` in query files |
| Prisma | `prisma\.\w+\.(findMany|findFirst|findUnique|count|aggregate|groupBy|create|createMany|update|updateMany|upsert|delete|deleteMany)`, `\$queryRaw`, `\$executeRaw`, `\$transaction` |
| TypeORM, Sequelize, Knex, Drizzle, Kysely | `getRepository`, `createQueryBuilder`, `\.find(One|AndCount)?\(`, `findAll`, `findAndCountAll`, `knex\(`, `db\.select\(`, `selectFrom\(` |
| Node raw | `\.query\(`, `\.execute\(` on a `pg`/`mysql2` client or pool |
| JPA, Spring Data, JDBC, jOOQ, MyBatis | `@Query`, repository interfaces (`findBy...`, `existsBy...`, `countBy...`), `createQuery`, `createNativeQuery`, `JdbcTemplate`, `dsl\.select`, mapper XML |
| Django | `\.objects\.`, `\.filter\(`, `\.raw\(`, `cursor\.execute`, and relation access in templates and serializers |
| SQLAlchemy | `session\.(execute|query|scalars|get)`, `select\(` |
| ActiveRecord | `\.where\(`, `\.find_by`, `\.includes\(`, `\.joins\(`, `find_by_sql`, `connection\.execute` |
| Eloquent, Doctrine | `DB::`, `::with\(`, `->where\(`, `createQueryBuilder`, DQL strings |
| EF Core, Dapper | LINQ on a `DbSet`, `\.Include\(`, `FromSqlRaw`, `\.Query<` |
| MongoDB | `\.find\(`, `\.aggregate\(`, `\.update(One|Many)\(`, `populate\(` |
| In the database | `CREATE FUNCTION`, `CREATE PROCEDURE`, `CREATE TRIGGER`, `CALL`, materialized view refreshes |

Also look where queries hide: template loops, serializers, GraphQL field resolvers, model
hooks and callbacks, ORM default scopes, and middleware that loads the current user or tenant
on every request.

## 4. Resolving the SQL that actually runs

Analyze the query the database receives, not the method name. Label the SQL text itself:

| How the query is written | Label for the SQL text |
|---|---|
| Literal SQL in code, a sqlc query file, a MyBatis mapper, JPA `nativeQuery` | `CONFIRMED` |
| SQL assembled by string building or conditional clauses | `CONFIRMED` per branch; list the branches that change filters, joins, or ordering |
| Query builder (squirrel, Knex, jOOQ, Kysely, Drizzle) | clauses `CONFIRMED` from the builder calls; exact text `INFERRED` |
| ORM call (GORM, Prisma, Hibernate, Django, ActiveRecord, ...) | the call and its arguments `CONFIRMED`; the generated SQL `INFERRED` from that ORM version's documented behavior; exact SQL `UNKNOWN` until captured (see `orm-performance.md`) |
| Captured from a SQL log, APM trace, `pg_stat_statements`, or `performance_schema` | `RUNTIME` |

Resolve the dynamic parts: optional filters (each combination is a different query shape for
the index analysis), user-selected sort columns (is there an allowlist?), page size and page
depth bounds, IN-list sources and their bounds, and hidden predicates added by the ORM
(soft-delete scopes, tenant scopes).

## 5. Connecting a query to its execution path

A query matters only through the code that runs it.

1. Find every caller of the data-access function. Resolve interfaces to the implementation
   wired into the process (see how `main`, the DI container, or the constructor graph builds
   it).
2. Walk callers up to an entry point: route registration, RPC registration, consumer
   subscription, scheduler, CLI command, or startup.
3. Record the path as `entry -> handler -> service -> repository -> query`, one symbol per hop.
4. Record the enclosing context on the path: loops (with what they iterate), transactions,
   locks, cache lookups before the query, and concurrency (`go`, `Promise.all`, `errgroup`,
   parallel streams).
5. Confirm reachability. A repository method with no caller, a route that is not registered,
   or a job that is not scheduled is not a performance risk. Mention it only if documentation
   claims it runs, or if the user asked for an inventory of all queries.

If an MDW `distributed-flow` or `sequence-flow` artifact for the entry point exists (by default
under `mdw/distributed-flow/` or `mdw/sequence-flow/`), use its path as a map and verify the hops
you rely on in code. Do not re-document the flow; record only the database interactions on it.

## 6. Frequency and cost driver

Performance impact is frequency times cost. Static analysis cannot measure either, but it can
classify both from code structure. Do not translate either into numbers.

**Frequency class** (how often the query runs):

| Class | Meaning | Evidence |
|---|---|---|
| `PER_REQUEST` | once per call of an entry point | on the handler's path, outside any loop |
| `PER_ITEM` | once per element of a runtime collection; multiplies its parent's frequency by N | loop, `map`, `Promise.all`, list comprehension, per-field resolver |
| `PER_MESSAGE` | once per consumed message or job | consumer or worker handler |
| `PER_RUN` | once per scheduled run; state the schedule and how many instances run it | cron, ticker, scheduler, plus replica count from manifests |
| `STARTUP` | at boot or migration | `main`, init hooks |
| `RARE` | admin, back-office, manual command; say why you think it is rare | route group, CLI, feature flag |
| `UNKNOWN` | caller not found or dispatch is dynamic | |

**Cost driver** (what the query's work grows with):

| Driver | Work grows with | Typical evidence |
|---|---|---|
| `CONSTANT` | nothing; a lookup by primary key or unique index | `WHERE id = $1`, unique constraint |
| `RESULT` | rows returned, bounded by a limit | `LIMIT` with an index that serves filter and order |
| `KEY_FANOUT` | rows per parent key (history per account) | `WHERE account_id = $1` without `LIMIT`, or with a sort the index cannot serve |
| `TABLE` | total table size | no filter, non-sargable predicate, sort or aggregate over the whole table, `COUNT(*)` of large sets |
| `OFFSET` | page depth | `OFFSET` from user input |
| `UNKNOWN` | depends on a plan or data distribution you cannot see | |

Combine them: `PER_ITEM` x `TABLE` (a scan repeated N times) is a much stronger static signal
than either alone. Write the combination in the finding.

Table size is almost always `UNKNOWN`. Growth signals make a conditional risk more likely but
never give a size: append-only tables (events, audit logs, history, messages, orders, invoices),
per-user or per-tenant records, and tables with retention jobs. A lookup table of countries or
settings stays small.

## 7. The query inventory

While reading, keep one line per query so nothing has to be re-read:

```text
Q7  SELECT * FROM invoices WHERE account_id = $1 AND deleted_at IS NULL ORDER BY issued_at DESC LIMIT $2 OFFSET $3
    internal/billing/repo.go InvoiceRepo.ListByAccount
    GET /accounts/{id}/invoices -> InvoiceHandler.List -> InvoiceService.ListForAccount
    tables invoices   freq PER_REQUEST   driver OFFSET + KEY_FANOUT   tx none   loop none
    notes  SELECT * includes pdf BYTEA; offset from ?page (no max)                 CONFIRMED
```

In the artifact, the inventory is a table (columns in `report-format.md`). Every row names the
database, the location, and the entry point, or `UNKNOWN` with what would resolve it.

## 8. Large repositories: choosing what to analyze

Count every query cheaply by search; analyze deeply only the ones that can matter. Deep-dive, in
this order:

1. queries on the paths the user asked about
2. queries inside loops or per-item fan-out
3. queries inside transactions that also lock rows or call out of the process
4. queries on tables with growth signals whose predicate or ordering is not served by an index
5. unbounded reads (`findMany`/`Find`/`SELECT` without `LIMIT`) on request paths
6. every scheduled job that iterates a table

State the numbers in Scope ("142 query call sites found; 31 analyzed; selection: request paths
of the order and billing APIs, all jobs, all queries in loops"). An unanalyzed query is not a
clean query.
