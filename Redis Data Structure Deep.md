# Redis Data Structure Deep dev

For each structure, I will break down:

1. **The "Why"**: The specific problem it solves and its internal mechanics.
2. **The "How"**: Detailed CLI syntax with **every argument explained separately**.
3. **Deep-Dive Examples**: Real-world scenarios showing exactly how the arguments interact.

---

### 1. Strings

**Internal Mechanism:** Simple Dynamic String (SDS). It is a binary-safe buffer that can hold text, integers, or serialized objects.
**The "Why":** Use this for caching simple values, counters, or distributed locks. It is the most fundamental structure.

### Command: `SET`

**Syntax:**

```powershell
SET key value [EX seconds | PX milliseconds | EXAT timestamp | PXAT timestamp] [NX | XX] [KEEPTTL] [GET]
```

**Argument Breakdown:**

- `key`: The unique identifier.
- `value`: The data to store.
- `EX seconds`: **(Optional)** Sets the Time-To-Live (TTL) in seconds. The key is automatically deleted after this time.
- `PX milliseconds`: **(Optional)** Sets TTL in milliseconds. More precise than `EX`.
- `NX`: **(Optional)** "Not Exists". The command only succeeds if the key does **not** already exist. Returns `nil` if it exists. *Crucial for distributed locks.*
- `XX`: **(Optional)** "Exists". The command only succeeds if the key **already** exists. Useful for updating existing cache entries without creating new ones.
- `KEEPTTL`: **(Optional)** Retains the existing TTL of the key instead of resetting it.
- `GET`: **(Optional)** Returns the old value stored at the key before overwriting it.

**Deep-Dive Example: Distributed Lock**

```bash
# Attempt to acquire a lock on 'resource:A' for 10 seconds.
# NX ensures only one client gets the lock.
# EX ensures the lock expires if the client crashes (prevents deadlocks).
127.0.0.1:6379> SET lock:resource:A "client_1_id" NX EX 10
OK
# Success! Client 1 has the lock.

# Another client tries immediately:
127.0.0.1:6379> SET lock:resource:A "client_2_id" NX EX 10
(nil)
# Failed. Key exists. Client 2 must retry or wait.
```

### Command: `INCR` / `INCRBY`

**The "Why":** Atomic integer operations. Thread-safe without needing application-level locking.
**Syntax:**

```bash
INCR key
INCRBY key increment
```

**Argument Breakdown:**

- `key`: Must hold an integer value (or be empty, treated as 0).
- `increment`: Integer amount to add. Can be negative for decrementing.

**Deep-Dive Example: Rate Limiting Counter**

```bash
127.0.0.1:6379> SET rate_limit:user:101 0 EX 60
OK
127.0.0.1:6379> INCR rate_limit:user:101
(integer) 1
127.0.0.1:6379> INCRBY rate_limit:user:101 5
(integer) 6
```

---

### 2. Lists

**Internal Mechanism:** Doubly Linked List.
**The "Why":** Ordered sequence of strings. O(1) insertion/removal at heads/tails. Ideal for Message Queues (FIFO) or Stacks (LIFO).

### Command: `LPUSH` / `RPUSH`

**Syntax:**

```bash
LPUSH key element [element ...]
RPUSH key element [element ...]
```

**Argument Breakdown:**

- `key`: The list identifier.
- `element`: One or more values to push.
    - `LPUSH`: Adds to the **head** (left). Last element added is the first one retrieved by `LPOP`.
    - `RPUSH`: Adds to the **tail** (right). First element added is the first one retrieved by `LPOP`.

### Command: `BLPOP` / `BRPOP` (Blocking)

**The "Why":** Efficient consumer pattern. Instead of polling (checking repeatedly), the client blocks until data arrives.
**Syntax:**

```bash
BLPOP key [key ...] timeout
```

**Argument Breakdown:**

- `key [key ...]`: You can listen to multiple lists. Redis checks them in order.
- `timeout`: Seconds to wait. `0` means wait indefinitely. If timeout hits, returns `nil`.

**Deep-Dive Example: Job Queue**

```bash
# Producer: Push jobs to the tail
127.0.0.1:6379> RPUSH job_queue "email_job_1" "email_job_2"
(integer) 2

# Consumer: Blockingly wait for a job from the head
# If queue is empty, waits up to 5 seconds
127.0.0.1:6379> BLPOP job_queue 5
1) "job_queue"
2) "email_job_1"
# Returns the key name and the value.
```

### Command: `LTRIM`

**The "Why":** Keep lists bounded in memory. Essential for "Recent Activity" feeds.
**Syntax:**

```bash
LTRIM key start stop
```

**Argument Breakdown:**

- `start`: Start index (0-based).
- `stop`: Stop index. `1` represents the last element.
- *Effect:* Deletes all elements outside the range `[start, stop]`.

**Deep-Dive Example: Keep Last 10 Logs**

```bash
127.0.0.1:6379> LPUSH logs "new_event"
(integer) 11 # Suppose list now has 11 items
127.0.0.1:6379> LTRIM logs 0 9
OK
# Items at index 10+ are deleted. List size is now 10.
```

---

### 3. Sets

**Internal Mechanism:** Hash Table.
**The "Why":** Unordered collection of **unique** values. O(1) complexity for add, remove, and check. Ideal for tags, unique visitors, and set mathematics (intersection/union).

### Command: `SADD`

**Syntax:**

```bash
SADD key member [member ...]
```

**Argument Breakdown:**

- `key`: The set identifier.
- `member`: Value to add. If it already exists, it is ignored. Returns the number of *new* elements added.

### Command: `SISMEMBER`

**Syntax:**

```bash
SISMEMBER key member
```

**Argument Breakdown:**

- `key`: The set identifier.
- `member`: Value to check.
- *Return:* `1` if exists, `0` if not.

### Command: `SINTER` (Intersection)

**The "Why":** Find common elements between sets. E.g., "Friends who like both Page A and Page B".
**Syntax:**

```bash
SINTER key [key ...]
```

**Argument Breakdown:**

- `key [key ...]`: Two or more set keys. Returns members present in **ALL** specified sets.

**Deep-Dive Example: Friend Recommendations**

```bash
# User 1 likes: Redis, Python
127.0.0.1:6379> SADD user:1:likes "Redis" "Python"
(integer) 2

# User 2 likes: Redis, Java
127.0.0.1:6379> SADD user:2:likes "Redis" "Java"
(integer) 2

# Find common interests
127.0.0.1:6379> SINTER user:1:likes user:2:likes
1) "Redis"
# Recommendation engine can suggest "Java" to User 1 based on this overlap.
```

---

### 4. Sorted Sets (ZSets)

**Internal Mechanism:** Skip List + Hash Table.
**The "Why":** Like a Set, but every member has a **score** (float). Elements are sorted by score. O(log N) complexity. Ideal for Leaderboards, Priority Queues, and Time-Series.

### Command: `ZADD`

**Syntax:**

```bash
ZADD key [NX | XX] [CH] [INCR] score member [score member ...]
```

**Argument Breakdown:**

- `key`: The sorted set identifier.
- `NX`: Only add new elements. Do not update scores of existing elements.
- `XX`: Only update scores of existing elements. Do not add new ones.
- `CH`: "Changed". Return the number of changed elements (added + score updated) instead of just new elements.
- `INCR`: Increment the score of the member by the provided value (like `ZINCRBY`).
- `score`: Floating point number determining sort order.
- `member`: The unique string value.

### Command: `ZRANGE` / `ZREVRANGE`

**The "Why":** Retrieve elements by rank (position) or score range.
**Syntax:**

```bash
ZRANGE key min max [BYSCORE | BYLEX] [REV] [LIMIT offset count] [WITHSCORES]
```

**Argument Breakdown:**

- `min`: Start index (if by rank) or min score (if `BYSCORE`).
- `max`: Stop index or max score.
- `BYSCORE`: Interpret min/max as scores, not indices.
- `REV`: Reverse the order (highest to lowest).
- `WITHSCORES`: Return the score along with the member.

**Deep-Dive Example: Gaming Leaderboard**

```bash
# Add players with scores
127.0.0.1:6379> ZADD leaderboard 100 "Alice" 200 "Bob" 150 "Charlie"
(integer) 3

# Update Alice's score (Increment by 50)
127.0.0.1:6379> ZADD leaderboard INCR 50 "Alice"
"150"

# Get Top 2 Players (Highest Score First)
# ZREVRANGE is shorthand for ZRANGE ... REV
127.0.0.1:6379> ZREVRANGE leaderboard 0 1 WITHSCORES
1) "Bob"
2) "200"
3) "Alice"
4) "150"
# Charlie (150) is tied with Alice, but Bob is higher.
```

---

### 5. Hashes

**Internal Mechanism:** ZipList (small) -> Hash Table (large).
**The "Why":** Represents objects with fields. More memory-efficient than storing a JSON string because you can update individual fields without rewriting the whole object.

### Command: `HSET`

**Syntax:**

```bash
HSET key field value [field value ...]
```

**Argument Breakdown:**

- `key`: The hash identifier (e.g., `user:101`).
- `field`: The attribute name (e.g., `name`).
- `value`: The attribute value.
- *Note:* You can set multiple field-value pairs in one command.

### Command: `HINCRBY`

**The "Why":** Atomically increment a numeric field within a hash.
**Syntax:**

```bash
HINCRBY key field increment
```

**Argument Breakdown:**

- `key`: The hash identifier.
- `field`: The specific numeric field to increment.
- `increment`: Integer amount.

**Deep-Dive Example: User Profile**

```bash
# Set initial profile
127.0.0.1:6379> HSET user:101 name "Muhammed" age "30" login_count "0"
(integer) 3

# Update only the name (efficient, doesn't touch age/login_count)
127.0.0.1:6379> HSET user:101 name "Muhammed Ali"
(integer) 0 # 0 means field was updated, not newly created

# Increment login count atomically
127.0.0.1:6379> HINCRBY user:101 login_count 1
(integer) 1

# Get specific fields
127.0.0.1:6379> HMGET user:101 name login_count
1) "Muhammed Ali"
2) "1"
```

---

### 6. Bitmaps

**Internal Mechanism:** Bit-level operations on String values.
**The "Why":** Extremely memory-efficient boolean arrays. 1 bit per user. 1 million users = ~125KB. Ideal for Daily Active Users (DAU).

### Command: `SETBIT`

**Syntax:**

```bash
SETBIT key offset value
```

**Argument Breakdown:**

- `key`: The bitmap identifier (e.g., `dau:2026-04-14`).
- `offset`: The bit position (usually User ID). Must be >= 0.
- `value`: `0` or `1`.

### Command: `BITCOUNT`

**Syntax:**

```bash
BITCOUNT key [start end [BYTE | BIT]]
```

**Argument Breakdown:**

- `key`: The bitmap identifier.
- `start/end`: Optional byte range to count within.
- *Return:* Number of bits set to 1.

**Deep-Dive Example: Daily Active Users**

```bash
# User ID 1001 logs in
127.0.0.1:6379> SETBIT dau:2026-04-14 1001 1
(integer) 0 # Previous value was 0

# User ID 1002 logs in
127.0.0.1:6379> SETBIT dau:2026-04-14 1002 1
(integer) 0

# Count total active users
127.0.0.1:6379> BITCOUNT dau:2026-04-14
(integer) 2
```

---

### 7. HyperLogLog

**Internal Mechanism:** Probabilistic counting algorithm.
**The "Why":** Count unique elements with fixed memory (~12KB) regardless of cardinality. Error rate ~0.81%. Use when exact count isn't critical but memory is.

### Command: `PFADD`

**Syntax:**

```bash
PFADD key element [element ...]
```

**Argument Breakdown:**

- `key`: The HLL identifier.
- `element`: The item to count (e.g., IP address).

### Command: `PFCOUNT`

**Syntax:**

```bash
PFCOUNT key [key ...]
```

**Argument Breakdown:**

- `key [key ...]`: One or more HLL keys. If multiple, it returns the union cardinality.

**Deep-Dive Example: Unique Visitors**

```bash
127.0.0.1:6379> PFADD visitors:today "ip_1" "ip_2" "ip_3"
(integer) 1

127.0.0.1:6379> PFADD visitors:today "ip_1"
(integer) 0 # Duplicate ignored

127.0.0.1:6379> PFCOUNT visitors:today
(integer) 3
```

---

### 8. Geospatial Indexes (Geo)

**Internal Mechanism:** Sorted Set with Geohash encoding.
**The "Why":** Store latitude/longitude and perform radius searches.

### Command: `GEOADD`

**Syntax:**

```bash
GEOADD key [NX | XX] [CH] longitude latitude member [longitude latitude member ...]
```

**Argument Breakdown:**

- `longitude`: X coordinate (-180 to 180).
- `latitude`: Y coordinate (-85.05112878 to 85.05112878).
- `member`: Name of the location.

### Command: `GEORADIUS` / `GEOSEARCH`

**The "Why":** Find items within a distance.
**Syntax:**

```bash
GEORADIUS key longitude latitude radius M|KM|FT|MI [WITHCOORD] [WITHDIST] [COUNT count] [ASC|DESC]
```

**Argument Breakdown:**

- `longitude/latitude`: Center point of search.
- `radius`: Distance radius.
- `M|KM|FT|MI`: Unit of measurement.
- `WITHCOORD`: Return lat/lon of results.
- `WITHDIST`: Return distance from center.
- `COUNT`: Limit number of results.

**Deep-Dive Example: Nearby Stores**

```bash
127.0.0.1:6379> GEOADD stores 13.36 38.11 "StoreA" 15.08 37.50 "StoreB"
(integer) 2

# Find stores within 100km of lon 14, lat 38
127.0.0.1:6379> GEORADIUS stores 14 38 100 KM WITHDIST
1) 1) "StoreA"
   2) "45.2"  # 45.2 km away
2) 1) "StoreB"
   2) "98.1"  # 98.1 km away
```

---

### 9. Streams

**Internal Mechanism:** Radix Tree. Append-only log.
**The "Why":** Robust message queuing with consumer groups, acknowledgments, and history replay. Superior to Lists for complex event processing.

### Command: `XADD`

**Syntax:**

```bash
XADD key [NOMKSTREAM] [MAXLEN [=|~] threshold] * field value [field value ...]
```

**Argument Breakdown:**

- `key`: Stream identifier.
- `MAXLEN`: Trim stream to max length. `~` makes it approximate (faster).
- : Auto-generate ID (timestamp-sequence).
- `field value`: Pairs of data.

### Command: `XREADGROUP`

**The "Why":** Read messages as part of a consumer group. Ensures each message is processed by only one consumer in the group.
**Syntax:**

```bash
XREADGROUP GROUP group consumer [COUNT count] [BLOCK ms] STREAMS key [key ...] id
```

**Argument Breakdown:**

- `GROUP`: Keyword.
- `group`: Name of the consumer group.
- `consumer`: Name of this specific worker.
- `STREAMS`: Keyword.
- `key`: Stream name.
- `id`: ID to read from. Use `>` to read only **new** messages assigned to this group.

**Deep-Dive Example: Event Processing**

```bash
# 1. Create Group
127.0.0.1:6379> XGROUP CREATE mystream mygroup $ MKSTREAM
OK

# 2. Add Event
127.0.0.1:6379> XADD mystream * type "order" id "123"
"1713123456789-0"

# 3. Consumer Reads New Event
127.0.0.1:6379> XREADGROUP GROUP mygroup worker1 STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) "1713123456789-0"
         2) 1) "type"
            2) "order"
            3) "id"
            4) "123"

# 4. Acknowledge (Remove from Pending Entries List)
127.0.0.1:6379> XACK mystream mygroup 1713123456789-0
(integer) 1
```

This structure provides the **Why**, **How**, and **Argument Details** for every major Redis data structure, enabling you to use them with precision in your CLI workflows.