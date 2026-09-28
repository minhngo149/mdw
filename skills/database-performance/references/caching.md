# Caching and the Database

How to inventory the caches in front of the database, check what each one saves and what it
risks, find repeated queries, and decide when a stampede is a supported risk rather than a
generic worry. A cache is never a default recommendation. It trades consistency, memory, and
complexity for latency, and it hides query problems instead of fixing them.

## Contents

1. Cache inventory
2. What to check for each cache
3. Invalidation
4. Stampede
5. Repeated queries
6. When a cache is not the fix
7. Trade-offs
8. Validation

## 1. Cache inventory

Find caches by their clients and APIs:

| Kind | Evidence |
|---|---|
| Remote cache | Redis clients (go-redis, ioredis, node-redis, redis-py, Jedis, Lettuce, Redisson, StackExchange.Redis, predis, phpredis), Memcached clients |
| In-process cache | Go `sync.Map` or a map with a mutex, `ristretto`, `bigcache`, `golang-lru`; Node `lru-cache`; Java Caffeine, Guava; Python `functools.lru_cache`/`cache`, `cachetools` |
| Framework cache | Spring `@Cacheable`/`@CacheEvict`/`@CachePut`, Django cache framework and `@cache_page`, Rails `Rails.cache.fetch`, Laravel `Cache::remember` |
| ORM cache | Hibernate second-level and query caches, and their providers |
| HTTP cache | `Cache-Control`, `ETag`, `Last-Modified` handling, CDN configuration in the repository |
| Request-scoped memoization | DataLoader, a per-request map, `@RequestScope` beans |

Record each one:

```text
CACHE account-summary   Redis   cache-aside                           internal/accounts/service.go Summary
  key         account:summary:{account_id}          ttl 10 min (fixed)
  value       aggregate of invoices and payments    source query Q12 (SUM over invoices by account)
  miss        runs Q12, then SET                    error: GET failure ignored -> runs Q12 (fail-open)
  writers     InvoiceService.Create, PaymentService.Apply (write the source tables)
  invalidates none found                                                               CONFIRMED
  coalescing  none found                                                               CONFIRMED
```

## 2. What to check for each cache

| Check | What goes wrong |
|---|---|
| Does the key include every input that changes the result (tenant, locale, permissions, filters, page)? | users see the wrong data (correctness) |
| Key cardinality: fixed, per entity, per query string? | per-query-string keys have low hit rates and grow memory without bound |
| TTL: none, fixed, jittered? | none: lives until eviction. Fixed TTL on keys populated together: they expire together |
| Is the expensive query cached, or a cheap one next to it? | the cache saves little |
| Are misses for missing entities cached (negative caching)? | repeated lookups of absent keys always reach the database |
| Error behavior: fail-open (query the database) or fail-closed? | fail-open moves all cached load onto the database when the cache is down |
| In-process cache with several instances | each instance has its own copy; invalidation in one does not reach the others; memory times instances |
| Value size and serialization | large values cost serialization and network on every hit |

## 3. Invalidation

Find every writer of the data behind each cached value: search for `INSERT`, `UPDATE`, and
`DELETE` on the source tables, and for ORM saves of the source models. Then check whether each
writer deletes or updates the key.

- A writer that does not invalidate means stale reads for up to the TTL. That is a correctness
  finding. Report it next to the cache, with the TTL as the staleness bound, and do not rate it
  as a performance finding.
- Write order matters. Deleting the key and then writing the database lets a concurrent reader
  re-cache the old value. Writing the database and then deleting the key has a smaller window.
- Write-through or dual writes (database and cache updated separately) can diverge when one
  write fails.

## 4. Stampede

A stampede (thundering herd, dog-piling) is many concurrent misses on the same key, each
recomputing the same expensive value. Report it as a risk only when the code shows **all** of:

1. a shared key that many concurrent requests read: a global key, or a per-entity key on an
   entity that most traffic targets
2. a moment when it misses for everyone at once: a fixed TTL, an explicit delete, or eviction
3. an expensive recompute: an aggregate, a scan, several queries, or a remote call
4. no coalescing mechanism

Coalescing mechanisms to look for: Go `golang.org/x/sync/singleflight`, groupcache; a
distributed lock around the recompute (Redis `SET NX`); stale-while-revalidate or soft TTLs;
probabilistic early refresh; a background refresher that keeps the key warm; Caffeine
`LoadingCache`/`get(key, fn)` (one load per key per process); Spring `@Cacheable(sync = true)`
(per process); Rails `race_condition_ttl`. In-process coalescing still allows one recompute per
instance.

Per-entity keys with little concurrency per entity rarely stampede. Say so, or leave them out.
Traffic is `UNKNOWN` from the repository, so a supported stampede risk is still conditional:
"conditional on concurrent traffic to `GET /reports/summary`".

## 5. Repeated queries

Look for the same query running more than once for one logical operation:

- the current user or tenant loaded in middleware and again in the handler
- configuration or reference data loaded inside a loop
- the same entity loaded by several services on one request path
- a lookup repeated per item for items that share the same key

The first fix is usually structural: load once and pass it down, or use a request-scoped memo.
A shared cache is the second option.

## 6. When a cache is not the fix

- The query is slow because of a missing index, an N+1, or an unbounded read: fix that first. A
  cache over it still pays the full cost on every miss.
- The data needs read-your-writes consistency (balances, permissions, inventory at checkout).
- Reuse is low: keys are unique per request.
- The data changes more often than it is read.
- The query is a primary-key lookup on a hot, small table that the database's buffer cache
  already holds (`INFERRED`; confirm with buffer hit statistics).

## 7. Trade-offs

State these with any cache recommendation:

| Dimension | Cost |
|---|---|
| Latency | hits are faster; misses are slower by one cache round trip |
| Consistency | stale reads up to the TTL; invalidation races |
| Memory | cache memory, the eviction policy, per-instance copies |
| Invalidation complexity | every writer of the source data must be found and kept correct |
| Stampede | new failure mode at expiry (section 4) |
| Availability | a new dependency; decide fail-open or fail-closed |
| Cold start | deploys and flushes send the full load to the database |

## 8. Validation

- Hit and miss rates per cache or key prefix. Redis `INFO stats` (`keyspace_hits`,
  `keyspace_misses`) is server-wide, so per-cache rates need application metrics.
- Database statements per request with a warm cache and a cold cache.
- Latency distribution of hits and misses separately.
- For a suspected stampede: in staging, expire the key under concurrent load and count how many
  times the recompute query runs (`pg_stat_statements.calls` delta, `performance_schema` digest
  counts, or application logs).
