# 12. Common Pitfalls & Anti-Patterns

As a senior engineer, it's not enough to know how to use a tool; you must also understand its limitations and common misuses. Avoiding these pitfalls will save you from significant headaches in production.

### Overusing Redis

**Why it's bad**: Redis is incredibly fast, but it's not a silver bullet. Using it for every data storage need can lead to increased operational complexity, higher costs, and using the wrong tool for the job.

**Example**: Storing all application data, including complex relational data, in Redis instead of a proper relational database. This often leads to reinventing the wheel for transactions, complex queries, and data integrity.

**When NOT to use**: For primary storage of highly structured, relational data that requires ACID transactions and complex querying capabilities. For long-term archival storage.

### Storing Large Objects

**Why it's bad**: Redis is optimized for small, frequently accessed data. Storing very large objects (e.g., multi-MB JSON strings, entire images) can lead to:

- **Memory Bloat**: Rapidly consumes RAM, leading to evictions or OOM errors.
- **Performance Degradation**: Large objects take longer to transfer over the network and process, increasing latency for all clients.
- **Serialization/Deserialization Overhead**: Application-side overhead for converting large objects to/from strings.

**Example**: Storing a 10MB user profile JSON string in a single Redis key.

**Better Approach**: Store references to large objects (e.g., S3 URLs) in Redis, or break down large objects into smaller, manageable pieces using Redis Hashes.

### Misusing Persistence

**Why it's bad**: Incorrectly configuring or misunderstanding Redis persistence can lead to data loss or performance issues.

**Example**: Running Redis without any persistence (RDB or AOF) in a production environment where data durability is required. Or, using `appendfsync no` for AOF when high durability is critical.

**Better Approach**: Understand the trade-offs between RDB and AOF. For most production systems, a hybrid approach (AOF with RDB snapshots) is recommended. Ensure regular backups of RDB files.

### Bad Key Design

**Why it's bad**: A well-designed key schema is crucial for maintainability, performance, and memory efficiency. Poor key design can lead to:

- **Lack of Organization**: Difficult to understand what data a key represents.
- **Inefficient Operations**: Inability to use multi-key commands or patterns effectively.
- **Memory Waste**: Long, verbose key names consume more memory.

**Example**: `user_profile_data_for_user_with_id_12345` instead of `user:12345:profile`.

**Better Approach**: Use a consistent naming convention (e.g., `object_type:id:field`). Keep keys concise. Use Hashes to group related fields for an object under a single key.