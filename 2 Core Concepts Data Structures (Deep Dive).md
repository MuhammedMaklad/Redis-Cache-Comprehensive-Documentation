# 2. Core Concepts: Data Structures (Deep Dive)

**Redis** is often called **a data structure server** because it doesn't just store strings; it supports a rich set of data types. Understanding these data structures is fundamental to leveraging Redis effectively. For each data structure, we'll explore its internal behavior, time complexity, real-world applications, and example commands.

### Strings

**Description**: The most basic Redis data type. **Redis** Strings are binary-safe, meaning they can hold any kind of data, from a JPEG image to a serialized Ruby object. They are typically used for caching HTML fragments, storing simple key-value pairs, or managing counters.

**Internal Behavior**: Redis ***Strings are dynamically sized and can hold up to 512 MB***. When a string is small, Redis uses a ***compact representation to save memory***. As the string grows, Redis automatically reallocates memory to accommodate the new size. This dynamic resizing is optimized to minimize overhead.

**Time Complexity (Big-O)**:

- `GET`: O(1) - Constant time, as it's a direct lookup by key.
- `SET`: O(1) - Constant time, assuming no reallocation is needed. If reallocation occurs, it can be O(N) in the worst case, but amortized to O(1).
- `INCR`: O(1) - Atomic increment operation.

**Real-World Use Case**: **User Session Management**. Storing a user's session token or basic profile information. For example, `user:100:session` could hold a JSON string of session data.

**Example Commands**:

```bash
SET user:100:name "Alice"
GET user:100:name
INCR page_views
DECR daily_downloads
SETEX cache:homepage 3600 "<html>...</html>" # Set with expiration of 1 hour
```

### Lists

**Description**: **Redis Lists** are ***ordered collections of strings***. They are *implemented as linked lists, which makes them efficient for adding elements to the head or tail*, even with many elements. *They are ideal for* implementing *queues*, *message brokers*, or *event logs*.

**Internal Behavior**: Redis Lists are implemented using a **doubly linked list** of strings. For small lists, Redis uses a more memory-efficient data structure called a **ziplist**. When a list grows beyond a certain threshold (configurable), it converts to a *regular linked list*. This adaptive behavior optimizes for both memory and performance.

**Time Complexity (Big-O)**:

- `LPUSH`/`RPUSH`: O(1) - Adding elements to either end is constant time.
- `LPOP`/`RPOP`: O(1) - Removing elements from either end is constant time.
- `LINDEX`: O(N) - Accessing an element by index requires traversing the list.
- `LRANGE`: O(S+N) - Where S is the start offset and N is the number of elements in the range.

**Real-World Use Case**: **Task Queues**. A background worker system can use a Redis List as a queue. Producers `LPUSH` tasks onto the list, and consumers `RPOP` (or `BRPOP` for blocking pop) tasks from the other end.

**Example Commands**:

```bash
LPUSH myqueue "task1" "task2"
RPUSH mylog "event:user_login:123" "event:product_view:456"
LPOP myqueue
LRANGE mylog 0 -1 # Get all elements in the log
BLPOP myqueue 0 # Blocking pop, waits indefinitely for an element
```

### Sets

**Description**: **Redis Sets** ***are unordered collections of unique strings***. They are useful for storing unique items, performing membership tests, and executing set operations like unions, intersections, and differences. Think of them as mathematical sets.

**Internal Behavior**: Redis Sets are implemented using a **hash table**. For small sets containing only integers, Redis uses an **intset** (integer set) for memory efficiency. Once the set contains non-integer strings or grows beyond a certain size, it converts to a hash table. This ensures O(1) average time complexity for most operations.

**Time Complexity (Big-O)**:

- `SADD`/`SREM`: O(1) average - Adding or removing a single element.
- `SISMEMBER`: O(1) average - Checking for membership.
- `SUNION`/`SINTER`: O(N*M) worst case, where N and M are the sizes of the sets. More practically, O(N) where N is the size of the smallest set for intersection and sum of sizes for union.

**Real-World Use Case**: **Unique Visitors Tracking**. Store unique user IDs for a given page or resource. `SADD page:article_id:visitors user_id`.

**Example Commands**:

```bash
SADD users:online "Alice" "Bob" "Charlie"
SISMEMBER users:online "Alice"
SREM users:online "Bob"
SMEMBERS users:online
SADD programming:languages "Python" "Java" "C++"
SADD web:technologies "HTML" "CSS" "JavaScript" "Python"
SINTER programming:languages web:technologies # Returns "Python"
```

### Sorted Sets (ZSETs)

**Description**: Redis Sorted Sets are collections of unique strings, similar to Sets, but where each member is associated with a **score**. The members are always kept sorted by their scores, allowing for efficient retrieval by score range or by rank. They are perfect for leaderboards, rate limiters, and range queries.

**Internal Behavior**: Sorted Sets are implemented using a combination of a **hash table** and a **skiplist**. The hash table maps members to their scores, providing O(1) average time complexity for score lookups. The skiplist maintains the sorted order of members by score, enabling efficient range queries and rank retrieval. For small sorted sets, Redis might use a **ziplist** initially.

**Time Complexity (Big-O)**:

- `ZADD`/`ZREM`: O(log N) - Adding or removing elements, due to skiplist insertion/deletion.
- `ZRANK`/`ZSCORE`: O(log N) - Retrieving rank or score.
- `ZRANGE`/`ZREVRANGE`: O(log N + M) - Where M is the number of elements returned.

**Real-World Use Case**: **Leaderboards**. Store game scores or user reputation points. `ZADD game:leaderboard 1000 "player:Alice" 1200 "player:Bob"`.

**Example Commands**:

```bash
ZADD game:scores 100 "player:Alice" 150 "player:Bob" 75 "player:Charlie"
ZRANGE game:scores 0 -1 WITHSCORES # Get all players and their scores, sorted ascending
ZREVRANGE game:scores 0 2 WITHSCORES # Get top 3 players and their scores, sorted descending
ZSCORE game:scores "player:Alice"
ZINCRBY game:scores 20 "player:Bob" # Increase Bob's score by 20
```

### Hashes

**Description**: **Redis Hashes** are *maps between string fields and string values*. ***They are ideal for representing objects, where each object has multiple fields and values***. This is more memory-efficient than storing each field as a separate Redis key.

**Internal Behavior**: Redis Hashes are implemented using a **hash table**. For small hashes, Redis uses a **ziplist** to save memory. When the hash grows beyond a certain number of elements or the size of its values exceeds a threshold, it converts to a regular hash table. This provides O(1) average time complexity for field access.

**Time Complexity (Big-O)**:

- `HSET`/`HGET`: O(1) average - Setting or getting a single field.
- `HMSET`/`HMGET`: O(N) average - Setting or getting multiple fields, where N is the number of fields.
- `HGETALL`: O(N) - Retrieving all fields and values.

**Real-World Use Case**: **Storing User Profiles**. Instead of `SET user:100:name "Alice"`, `SET user:100:email "alice@example.com"`, you can use `HSET user:100 name "Alice" email "alice@example.com"`.

**Example Commands**:

```bash
HSET user:100 name "Alice" email "alice@example.com" age 30
HGET user:100 name
HMGET user:100 name email
HGETALL user:100
```

### Bitmaps

**Description**: **Redis Bitmaps** *are not a standalone data type but a set of bit-oriented operations that can be performed on String values*. They treat a String as a sequence of bits, allowing you to set or clear individual bits, count set bits, or find the first set/clear bit. They are extremely memory-efficient for storing boolean information.

**Internal Behavior**: Bitmaps operate directly on the underlying binary representation of a Redis String. Each character in a string is 8 bits, so by manipulating bits at specific offsets, Redis can effectively store boolean arrays.

**Time Complexity (Big-O)**:

- `SETBIT`: O(1) - Setting a single bit. Potentially O(N) if the string needs to be expanded.
- `GETBIT`: O(1) - Getting a single bit.
- `BITCOUNT`: O(N) - Counting set bits, where N is the length of the string in bytes.

**Real-World Use Case**: **User Activity Tracking (e.g., daily active users)**. Each bit can represent a user, and its value (0 or 1) can indicate if the user was active on a specific day. For example, `SETBIT daily:active:2026-04-11 user_id 1`.

**Example Commands**:

```bash
SETBIT daily:active:2026-04-11 0 1 # User 0 active
SETBIT daily:active:2026-04-11 100 1 # User 100 active
GETBIT daily:active:2026-04-11 0
BITCOUNT daily:active:2026-04-11 # Count active users for the day
```

### HyperLogLog

**Description**: **Redis HyperLogLog** is a *probabilistic data structure used to estimate the number of unique items in a set, using very little memory.* It's designed for counting unique elements (cardinality) when accuracy is not critical, but memory usage is paramount. The error rate is typically around 0.81%.

**Internal Behavior**: HyperLogLog uses a probabilistic algorithm to estimate cardinality. It doesn't store individual elements but rather a compact representation of the observed hash values. This allows it to estimate the count of millions of unique items using only 12 KB of memory.

**Time Complexity (Big-O)**:

- `PFADD`: O(1) average - Adding an element.
- `PFCOUNT`: O(1) - Estimating the cardinality.
- `PFMERGE`: O(N) - Merging multiple HyperLogLogs.

**Real-World Use Case**: **Counting Unique Page Views**. If you need to know how many unique users visited a page, but don't need the exact list of users, HyperLogLog is perfect. `PFADD page:views:article_id user_id`.

**Example Commands**:

```bash
PFADD unique:visitors "Alice" "Bob" "Charlie"
PFADD unique:visitors "Bob" "David"
PFCOUNT unique:visitors # Returns 4 (Alice, Bob, Charlie, David)
```

### Streams

**Description**: **Redis Streams** are *an append-only data structure that models a log*. They are designed for use cases like event sourcing, message queues, and real-time data processing. Streams allow multiple consumers to read from the same stream, maintaining their own consumption progress.

**Internal Behavior**: Streams are implemented as a sequence of entries, where each entry has a unique ID and a set of field-value pairs. They support consumer groups, allowing a group of clients to cooperate in consuming a stream of messages, with each message being delivered to only one consumer in the group. This is a significant feature for building robust, scalable message processing systems.

**Time Complexity (Big-O)**:

- `XADD`: O(1) - Adding an entry.
- `XRANGE`: O(N) - Reading a range of entries.
- `XREAD`: O(N) - Reading from a stream.

**Real-World Use Case**: **Event Sourcing / Microservices Communication**. A microservice can `XADD` events to a stream, and other microservices can `XREAD` from it to react to those events. For example, an `order:events` stream could contain `order_created`, `order_updated`, `order_shipped` events.

**Example Commands**:

```
XADD mystream * sensor_id 123 temperature 25.5
XADD mystream * sensor_id 456 temperature 24.9
XRANGE mystream - +
XREAD COUNT 2 STREAMS mystream 0-0

XGROUP CREATE mystream mygroup 0-0 MKSTREAM # Create a consumer group
XREADGROUP GROUP mygroup consumer1 COUNT 1 STREAMS mystream > # Read from a consumer group
```

---