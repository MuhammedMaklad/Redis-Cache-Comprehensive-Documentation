# 5. Redis as a Cache (Critical Section)

Caching is arguably the most common use case for Redis. A cache sits between your application and the primary database, storing frequently accessed data to reduce latency and database load. However, implementing caching correctly requires understanding various patterns and potential pitfalls.

### Caching Patterns

### Cache-Aside (Lazy Loading)

This is the most common caching pattern. The application code is responsible for managing the cache.

1. **Read**: The application first checks the cache.
    - **Cache Hit**: If the data is found, it's returned immediately.
    - **Cache Miss**: If the data is not found, the application queries the primary database, stores the result in the cache, and then returns it.
2. **Write**: The application writes data to the primary database and then either updates or invalidates (deletes) the corresponding cache entry.

**Pros**: Simple to implement, resilient to cache failures (the application falls back to the database).
**Cons**: The first request for data always results in a cache miss (latency penalty).

### Write-Through

In this pattern, the application **writes data to the cache**, and the **cache synchronously writes the data to the primary database.**

**Pros**: Data in the cache is always up-to-date.
**Cons**: Write operations are slower because they must complete in both the cache and the database before returning to the application. Redis doesn't natively support this out-of-the-box for external databases; it requires application-level coordination or specialized middleware.

### Write-Back (Write-Behind)

The application **writes data only to the cache**. The **cache asynchronously writes the data to the primary database after a delay or when a certain condition is met**.

**Pros**: Extremely fast write operations.
**Cons**: High risk of data loss if the cache crashes before writing to the database. Complex to implement robustly.

### The Cache Invalidation Problem

"There are only two hard things in Computer Science: cache invalidation and naming things." - Phil Karlton.

**Cache invalidation** *is the process of removing or updating stale data in the cache when the underlying data in the primary database changes*. If not handled correctly, your application will serve outdated information.

**Strategies**:

- **Time-To-Live (TTL)**: *Set an expiration time for cache entries*. This is the simplest approach but means data might be stale until the TTL expires.
- **Event-Driven Invalidation**: *When data is updated in the database, publish an event (e.g., via Redis Pub/Sub or a message queue) that triggers the cache to be invalidated or updated.*

### Common Caching Pitfalls

### 1. Cache Stampede (Thundering Herd)

**Problem**: A highly popular cache key expires (or is evicted). Suddenly, thousands of concurrent requests hit the cache, find it empty (cache miss), and all simultaneously query the primary database to regenerate the data. This massive spike in load can overwhelm and crash the database.

**Solution**:

- **Mutex Locks (Distributed Locks)**: When a cache miss occurs, only one request is allowed to query the database and update the cache. Other requests wait for the lock to be released or poll the cache again.
- **Probabilistic Early Expiration (XFetch)**: Artificially expire the cache slightly before its actual TTL for a small percentage of requests, allowing one background thread to refresh it while others still get the slightly stale cached value.

### 2. Cache Penetration

**Problem**: Attackers (or buggy code) repeatedly request data that does not exist in the cache *and* does not exist in the primary database (e.g., querying for user ID `-1`). Since the database returns nothing, nothing is cached, and every subsequent request hits the database, potentially causing a Denial of Service (DoS).

**Solution**:

- **Cache Empty Results**: If a database query returns no result, cache that "empty" result (e.g., a special string or a short TTL) so subsequent requests for the same non-existent key hit the cache instead of the database.
- **Bloom Filters**: Use a Bloom filter (a space-efficient probabilistic data structure) to quickly check if a key *might* exist in the database before querying it. If the Bloom filter says "no," reject the request immediately.

### 3. Cache Avalanche

**Problem**: A large number of cache keys expire at the exact same time, or the Redis server itself crashes and restarts empty. This leads to a massive wave of cache misses, overwhelming the primary database.

**Solution**:

- **Add Jitter to TTLs**: Instead of setting exactly the same TTL for many keys (e.g., 1 hour), add a random jitter (e.g., 1 hour +/- 5 minutes). This spreads out the expirations over time.
- **High Availability**: Use Redis Sentinel or Redis Cluster to ensure the cache layer remains available even if a single node fails.
- **Warm Up the Cache**: On startup, proactively load frequently accessed data into the cache before serving user traffic.

**Diagram: Cache-Aside Pattern**

```
[Application]
     |
     | 1. Check Cache
     v
  [Redis] ---> (Hit) ---> Return Data
     |
     | (Miss)
     v
 2. Query DB
     |
     v
[Database] ---> Return Data
     |
     | 3. Store in Cache
     v
  [Redis]
```