# 9. Performance Optimization

Optimizing Redis performance involves careful consideration of memory usage, command patterns, and client-side interactions. Thinking like a senior engineer means not just using Redis, but using it *efficiently*.

### Memory Optimization Techniques

Since Redis is an in-memory store, managing memory is paramount. 

High memory usage can lead to swapping (which kills performance) or OOM (Out Of Memory) errors.

- **Use Efficient Data Structures**: Choose the right Redis data type for your use case. For example, use Hashes for objects instead of separate String keys for each field. Use Sorted Sets for leaderboards instead of Lists and sorting in application code.
- **Small Keys and Values**: Keep keys and values as small as possible. Shorten key names (e.g., `u:100:n` instead of `user:100:name`). Compress large string values if possible before storing them.
- **Hash, List, Set, Zset Encoding**: Redis uses memory-efficient encodings (like ziplists and intsets) for small data structures. By keeping your hashes, lists, sets, and sorted sets small, you can benefit from these optimizations. The thresholds for these encodings are configurable (`hash-max-ziplist-entries`, `list-max-ziplist-entries`, etc. in `redis.conf`).
- **Avoid Key Duplication**: Don't store the same data under multiple keys unless absolutely necessary.
- **Eviction Policies**: Configure an appropriate `maxmemory` and `maxmemory-policy` to ensure Redis can gracefully handle memory pressure by evicting less important data. (See below for more details).

### Eviction Policies

When Redis reaches its `maxmemory` limit, it must decide which keys to remove in order to free up space. This behavior is controlled by the `maxmemory-policy` setting in `redis.conf`.

| Policy Name | Description |
| --- | --- |
| `noeviction` | Returns an error when memory limit is reached and a write operation is attempted. No keys are evicted. |
| `allkeys-lru` | Evicts the least recently used (LRU) keys from all keys. |
| `volatile-lru` | Evicts the least recently used keys, but only among keys with an expiration (`TTL`) set. |
| `allkeys-lfu` | Evicts the least frequently used (LFU) keys from all keys. |
| `volatile-lfu` | Evicts the least frequently used keys among those with a `TTL`. |
| `allkeys-random` | Randomly evicts keys from all keys. |
| `volatile-random` | Randomly evicts keys only among keys with a `TTL`. |
| `volatile-ttl` | Evicts keys with the nearest expiration time (shortest remaining TTL). |

---

#### Choosing the Right Policy

- **LRU-based (`allkeys-lru`, `volatile-lru`)**
    
    Best for caching scenarios where recently accessed data is more likely to be reused.
    
- **LFU-based (`allkeys-lfu`, `volatile-lfu`)**
    
    Ideal for workloads with stable access patterns where frequently accessed data should persist longer.
    
- **TTL-based (`volatile-ttl`)**
    
    Useful when expiration time is meaningful (e.g., session management, time-sensitive data).
    
- **Random (`allkeys-random`, `volatile-random`)**
    
    Simple and fast, but less predictable—rarely used in production.
    
- **No Eviction (`noeviction`)**
    
    Suitable when data loss is unacceptable, and the application must handle memory errors explicitly.
    

---

#### Practical Example

```
maxmemory 512mb
maxmemory-policy allkeys-lru
```

**Explanation:**

- Limits Redis memory usage to 512 MB
- When memory is full, removes the least recently used keys across all data

---

#### Key Insight (Senior Engineer Perspective)

Eviction policy selection is not just a configuration detail—it directly reflects your **data access pattern** and **business requirements**:

- If Redis is used as a **cache** → prefer `allkeys-lru` or `allkeys-lfu`
- If Redis stores **critical data** → avoid eviction (`noeviction`) and scale instead
- If using **expiring data heavily** → leverage `volatile-*` policies

---

#### Trade-offs

While vertical scaling (increasing memory on a single node) is simpler to implement, it eventually hits physical and cost limits. Horizontal scaling (e.g., sharding or clustering with Redis Cluster) introduces operational complexity—such as data partitioning, rebalancing, and network overhead—but provides virtually unlimited scalability and better fault tolerance.

### Redis Cluster

Redis Cluster is the official solution for horizontal scaling and high availability. It allows your dataset to be automatically sharded across multiple Redis instances.

**How it works**:

- **Hash Slots**: The key space is divided into 16384 hash slots. Each master node in the cluster is responsible for a subset of these hash slots.
- **Key-to-Slot Mapping**: A simple hash function (`CRC16(key) % 16384`) determines which slot a key belongs to.
- **Redirection**: Clients are aware of the cluster topology. If a client sends a command for a key to the wrong node, that node responds with a `MOVED` redirection error, telling the client which node is responsible for that hash slot. Clients then update their internal mapping and retry the command on the correct node.
- **Replication**: Each master node can have one or more replica nodes. If a master fails, one of its replicas is automatically promoted to take its place, ensuring high availability for that part of the dataset.

**Failover**: When a master node fails, other nodes in the cluster detect its unavailability. If a majority of master nodes agree that a master is down (a
failover), one of its replicas is elected and promoted to master. This process is automatic and transparent to clients (after they update their routing tables).

**Consistency Trade-offs**: Redis Cluster is designed for **Availability** and **Partition Tolerance**. During a network partition, if a client is partitioned from a master but can still reach its replica, the replica might not be able to be promoted to master if the majority of masters are on the other side of the partition. This means writes might be temporarily unavailable. Additionally, during failover, there's a small window where data might be lost if the old master had un-replicated writes.

### Sharding

Sharding is the process of distributing data across multiple machines. Redis Cluster provides automatic sharding. Manual sharding can also be implemented at the application level, where the application logic decides which Redis instance to store or retrieve a key from. This offers more control but adds significant complexity to the application.