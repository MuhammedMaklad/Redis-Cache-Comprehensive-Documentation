# 4. Persistence Mechanisms

Redis provides **two main persistence option**s: ***RDB and AOF***. You can use them individually or combine them for a hybrid approach.

### RDB (Redis Database Snapshots)

**How it works internally**: **RDB persistence** ***creates point-in-time snapshots of your dataset at specified intervals***. 

Redis forks a *child process* to perform the snapshotting. The child process writes the dataset to a temporary RDB file and, once complete, replaces the old RDB file. This process is highly optimized and minimizes the impact on the main Redis thread.

**Pros**:

- **Compact**: RDB files are compact, single-file representations of your data, making them ideal for backups and disaster recovery.
- **Performance**: RDB maximizes Redis performance because the main thread only needs to fork a child process to do the disk I/O, without blocking client requests.
- **Faster Restarts**: Restarting Redis with a large dataset is significantly faster with RDB compared to AOF.

**Cons**:

- **Data Loss Potential**: Since snapshots are taken periodically (e.g., every 5 minutes), any data written after the last snapshot will be lost if Redis crashes.
- **Forking Overhead**: Forking a child process can be CPU-intensive and temporarily pause the server if the dataset is very large, especially on systems with limited memory or slow CPUs.

**When to use**:

- When you can tolerate some data loss (e.g., caching scenarios where data can be recomputed).
- For taking regular backups of your dataset.
- When fast restart times are a priority.

**Configuration Example (`redis.conf`)**:

```
# Save the DB if both the given number of seconds and the given
# number of write operations against the DB occurred.
save 900 1      # Save after 900 sec (15 min) if at least 1 key changed
save 300 10     # Save after 300 sec (5 min) if at least 10 keys changed
save 60 10000   # Save after 60 sec if at least 10000 keys changed
```

### AOF (Append Only File)

**How it works internally**: **AOF persistence *logs every write operation received by the server.*** These operations are appended to the AOF file in the same format as the Redis protocol. When Redis restarts, it replays the AOF file to reconstruct the dataset. To prevent the AOF file from growing indefinitely, Redis can rewrite it in the background, creating a new, smaller AOF file that contains only the minimal set of commands needed to recreate the current dataset.

**Pros**:

- **High Durability**: AOF can be configured to `fsync` every second or even every query, minimizing data loss to almost zero.
- **Readable and Editable**: The AOF file is a text file containing Redis commands, making it possible to inspect or even edit it in emergencies (e.g., to remove an accidental `FLUSHALL` command).

**Cons**:

- **Larger File Size**: AOF files are generally larger than RDB files for the same dataset.
- **Slower Restarts**: Replaying the AOF file is slower than loading an RDB snapshot.
- **Performance Overhead**: Depending on the `fsync` policy, AOF can introduce more disk I/O overhead than RDB.

**When to use**:

- When data durability is critical and you cannot afford to lose even a few minutes of data.
- As a primary persistence mechanism, often in combination with RDB for backups.

**Configuration Example (`redis.conf`)**:

```
appendonly yes
# appendfsync always   # Very durable, but slow
appendfsync everysec # Good balance of durability and performance (default)
# appendfsync no       # Let the OS decide when to flush (fastest, least durable)
```

**Hybrid Persistence (Redis 4.0+)**:
Redis allows combining RDB and AOF. In this mode, the AOF rewrite process creates an RDB snapshot as the base of the new AOF file and then appends subsequent write commands. This provides the fast restart times of RDB and the high durability of AOF.