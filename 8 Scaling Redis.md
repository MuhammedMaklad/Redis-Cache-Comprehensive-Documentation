# 8. Scaling Redis

As your application grows, so does the demand on your data store. Scaling Redis effectively is crucial for maintaining performance and availability. There are two primary ways to scale: vertical and horizontal.

### Vertical vs. Horizontal Scaling

- **Vertical Scaling (Scale Up)**: ***Increasing the resources (CPU, RAM) of a single Redis server***. This is simpler but has limits (e.g., maximum RAM on a single machine) and introduces a single point of failure.
- **Horizontal Scaling (Scale Out)**: ***Distributing your data and load across multiple Redis instances.*** This offers greater scalability and fault tolerance but adds complexity.

### Replication (Master-Replica)

**Replication *is a fundamental building block for high availability and read scalability in Redis***. A master instance can have multiple replica instances.

**How it works**:

- **The master *handles all write operations.***
- **Replicas *connect to the master and receive a copy of the data***. ***They are read-only by default***.
- When a replica connects, it performs an initial synchronization (full resynchronization) by getting an RDB snapshot from the master. After that, it receives a continuous stream of write commands from the master.

**Benefits**:

- **High Availability**: *If the master fails*, a *replica can be promoted to master* (manually or automatically with Sentinel).
- **Read Scalability**: You can distribute read traffic across multiple replicas, offloading the master.
- **Data Redundancy**: Replicas hold copies of the data, protecting against data loss if the master fails.

**Trade-offs**:

- **Eventual Consistency**: Replicas might lag slightly behind the master, leading to eventual consistency. Reads from replicas might return slightly stale data.
- **Write Bottleneck**: All writes still go to a single master, limiting write scalability.

### Redis Sentinel

**Redis Sentinel** ***is a distributed system that provides high availability for Redis***. 

It monitors Redis master and replica instances, handles automatic failover if a master fails, and provides configuration to clients.

**Key functions**:

- **Monitoring**: Continuously checks if master and replica instances are working as expected.
- **Notification**: Notifies administrators or other computer programs when a Redis instance goes into an error condition.
- **Automatic Failover**: If a master is not working as expected, Sentinel can start a failover process where a replica is promoted to master, and other replicas are reconfigured to use the new master.
- **Configuration Provider**: Clients connect to Sentinel to discover the current master's address.

**Trade-offs**: Adds complexity to the deployment and requires a quorum of Sentinels to agree on a failover.

### Redis Cluster

**Redis Cluster** **provides a way to run a Redis installation where data is automatically sharded across multiple Redis nodes**.

It *offers horizontal scalability and high availability without requiring external coordination services like Sentinel* (though Sentinel can be used with Cluster for monitoring the cluster itself).

**How it works**:

- **Sharding**: ***The key space is divided into 16384 hash slots***. ***Each master node in the cluster is responsible for a subset of these hash slots***.
- **Redirection**: When a client sends a command for a key, the cluster determines which node owns the hash slot for that key. If the client sends the command to the wrong node, the node redirects the client to the correct node using a `MOVED` error.
- **Replication within Cluster**: Each master node can have its own replicas, providing high availability for that master's subset of data. If a master fails, one of its replicas is promoted.

**Benefits**:

- **Horizontal Scalability**: Distributes data and load across many nodes.
- **High Availability**: Automatic failover for master nodes.
- **Performance**: Can handle larger datasets and higher throughput than a single instance.

**Trade-offs**:

- **Complexity**: More complex to set up, manage, and operate than a standalone or master-replica setup.
- **Multi-key Operations**: Operations involving multiple keys must be performed on keys that reside in the same hash slot. If keys are in different slots, multi-key commands (like `MSET`, `SUNION`) are not directly supported and require client-side logic or Redis Lua scripting.
- **No Cross-Slot Transactions**: Transactions (MULTI/EXEC) are limited to a single hash slot.

### Consistency Trade-offs and CAP Theorem Relation

When scaling distributed systems, you inevitably encounter the **CAP theorem**, which states that a distributed data store can only provide two of the three guarantees:

- **Consistency (C)**: All clients see the same data at the same time.
- **Availability (A)**: Every request receives a response, without guarantee that it contains the most recent write.
- **Partition Tolerance (P)**: The system continues to operate despite network partitions (communication failures between nodes).

Redis, especially in a clustered or replicated setup, typically prioritizes **Availability** and **Partition Tolerance** over strong **Consistency**. For example, in a master-replica setup, if the master fails and a replica is promoted, there's a window where some data written to the old master might not have been replicated to the new master, leading to potential data loss or inconsistency. Redis Cluster is also AP by design.

**Implications**:

- Understand that in scaled Redis deployments, you might experience eventual consistency.
- Design your application to be tolerant of temporary inconsistencies or use mechanisms like distributed locks for critical operations requiring strong consistency.