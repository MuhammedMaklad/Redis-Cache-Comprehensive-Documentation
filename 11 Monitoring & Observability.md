# 11. Monitoring & Observability

Effective monitoring is essential for maintaining the health, performance, and stability of your Redis deployment. It allows you to detect issues early, troubleshoot problems, and optimize resource utilization.

### Redis CLI Tools

The Redis CLI (`redis-cli`) is your primary tool for interacting with Redis and gathering basic monitoring information.

- `INFO`: Provides a wealth of information about the Redis server, including memory usage, persistence status, replication, CPU usage, and client statistics. You can specify sections, e.g., `INFO memory`, `INFO replication`.
- `MONITOR`: Streams all commands processed by the Redis server in real-time. Useful for debugging but can be a performance overhead in production.
- `SLOWLOG GET [count]`: Retrieves entries from the slow log, which records commands that exceeded a configurable execution time threshold (`slowlog-log-slower-than`).
- `CLIENT LIST`: Shows a list of all connected client connections, including their IP, port, idle time, and flags.

### Metrics to Track

Beyond basic CLI tools, integrate Redis monitoring with your existing observability stack (Prometheus, Grafana, Datadog, etc.). Key metrics include:

- **Memory Usage**: `used_memory`, `used_memory_rss`, `mem_fragmentation_ratio`. High fragmentation can indicate inefficient memory usage.
- **CPU Usage**: `used_cpu_sys`, `used_cpu_user`. High CPU usage might point to expensive commands or insufficient resources.
- **Connections**: `connected_clients`, `blocked_clients`. A sudden spike in blocked clients can indicate a bottleneck.
- **Persistence**: `rdb_last_save_time`, `aof_last_rewrite_time_sec`, `aof_pending_bio_fsync`. Ensure persistence is working as expected.
- **Replication**: `master_link_status`, `master_last_io_seconds_ago`, `master_sync_in_progress`. For replicas, `master_repl_offset` and `slave_repl_offset` to check for replication lag.
- **Keyspace**: `db0:keys`, `expires`. Monitor the number of keys and expiring keys.
- **Latency**: `latency_sample_rate`. Track command execution times.
- **Evictions**: `evicted_keys`. If this number is high, your `maxmemory` might be too low or your eviction policy too aggressive.

### Slow Queries

Redis has a built-in slow log that records commands exceeding a specified execution time. Regularly review this log to identify and optimize problematic commands.

**Configuration (`redis.conf`)**:

```
slowlog-log-slower-than 10000 # Log commands slower than 10000 microseconds (10ms)
slowlog-max-len 128           # Keep a maximum of 128 entries in the slow log
```

**Identifying and Addressing Slow Queries**:

- **`KEYS` command**: Avoid `KEYS` in production; use `SCAN` for iterating over keys.
- **`LRANGE` on very long lists**: Use `LTRIM` to keep lists bounded.
- **Complex Lua scripts**: Profile and optimize your Lua scripts.
- **`SMEMBERS` on very large sets**: Consider using `SSCAN`.

### Memory Leaks Detection

While Redis itself is generally memory-safe, application-level issues can lead to Redis memory growth that resembles a leak.

- **Unbounded Data Structures**: Lists, Sets, Hashes, or Sorted Sets that grow indefinitely without proper cleanup (e.g., `LTRIM`, `SREM`, `HDEL`).
- **Keys without TTL**: Data that is written to Redis but never expires, leading to continuous memory consumption. Use `EXPIRE` or `SETEX` where appropriate.
- **Large Objects**: Storing excessively large strings or complex objects that consume a lot of memory.

Regularly monitor `used_memory_rss` and `mem_fragmentation_ratio`. If `used_memory_rss` grows unexpectedly and `mem_fragmentation_ratio` is high, it might indicate memory fragmentation or an application-level issue.