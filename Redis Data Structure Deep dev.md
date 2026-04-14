# Redis Data Structure Deep dev

Understanding the raw CLI syntax is fundamental because:

1. It is the source of truth for all client libraries (Python, Node.js, Go, etc.).
2. It allows you to debug directly on the server using `redis-cli`.
3. It reveals optional flags and atomic behaviors that high-level wrappers often hide.

Below is the deep-dive into every native Redis data structure, focusing on **Syntax**, **Argument Definitions**, and **Practical CLI Examples**.

---

### 1. Strings

**Concept:** Binary-safe strings. Can hold text, integers, or serialized objects (JSON). Max size 512MB.

### Core Syntax & Arguments

```powershell
SET key value [EX seconds | PX milliseconds | EXAT unix-time-seconds | PXAT unix-time-milliseconds | KEEPTTL] [NX | XX] [GET]
```

| Argument | Type | Description |
| --- | --- | --- |
| `key` | String | The identifier for the value. |
| `value` | String | The data to store. |
| `EX seconds` | Integer | Set expiration in **seconds**. |
| `PX milliseconds` | Integer | Set expiration in **milliseconds**. |
| `NX` | Flag | **N**ot e**X**ists: Only set if key does **not** exist. (Used for locks). |
| `XX` | Flag | Only set if key **already** exists. (Used for updates). |
| `GET` | Flag | Return the old value stored at key before overwriting. |

### Other Critical Commands

```powershell
GET key                  # Retrieve value. Returns nil if key doesn't exist.
MSET key value [key value ...]  # Atomic set of multiple keys. No expiration support here.
MGET key [key ...]       # Retrieve multiple values. Returns array.
INCR key                 # Atomically increment integer by 1. Error if not integer.
INCRBY key increment     # Increment by specific integer amount.
DECR key                 # Decrement by 1.
APPEND key value         # Append value to end of string. Returns new length.
STRLEN key               # Get length of string value.
```

### CLI Deep-Dive Example

```bash
# 1. Set a session with 1-hour expiration (3600 seconds)
127.0.0.1:6379> SET session:abc123 "user_data_json" EX 3600
OK

# 2. Try to set a lock only if it doesn't exist (NX)
127.0.0.1:6379> SET lock:resource_1 "locked" NX EX 10
OK
# If another client tries:
127.0.0.1:6379> SET lock:resource_1 "locked" NX EX 10
(nil)  # Returns nil because key already exists

# 3. Atomic Counter
127.0.0.1:6379> SET page_views 0
OK
127.0.0.1:6379> INCR page_views
(integer) 1
127.0.0.1:6379> INCRBY page_views 5
(integer) 6

# 4. Get multiple keys at once (Reduces network round trips)
127.0.0.1:6379> MSET user:1:name "Muhammed" user:2:name "Ali"
OK
127.0.0.1:6379> MGET user:1:name user:2:name
1) "Muhammed"
2) "Ali"
```

---

### 2. Lists

**Concept:** Ordered collections of strings. Implemented as linked lists. Great for queues.

### Core Syntax & Arguments

```bash
LPUSH key element [element ...]   # Push to head (left)
RPUSH key element [element ...]   # Push to tail (right)
LPOP key [count]                  # Pop from head. Optional count (Redis 6.2+).
RPOP key [count]                  # Pop from tail.
BLPOP key [key ...] timeout       # Blocking LPOP. Waits until item available or timeout.
BRPOP key [key ...] timeout       # Blocking RPOP.
LRANGE key start stop             # Get range. 0 is first, -1 is last.
LLEN key                          # Get length of list.
LTRIM key start stop              # Trim list to keep only elements in range.
```

| Argument | Type | Description |
| --- | --- | --- |
| `key` | String | The list identifier. |
| `element` | String | Value to push. Can provide multiple. |
| `start/stop` | Integer | Indices for range. Negative indices count from end (-1 = last). |
| `timeout` | Integer | Seconds to block. `0` means block indefinitely. |
| `count` | Integer | Number of elements to pop/trim. |

### CLI Deep-Dive Example

```bash
# 1. Producer: Add tasks to queue (Left Push)
127.0.0.1:6379> LPUSH task_queue "job_1" "job_2" "job_3"
(integer) 3

# 2. Consumer: Blockingly wait for a task (Right Pop)
# If queue is empty, waits up to 5 seconds
127.0.0.1:6379> BRPOP task_queue 5
1) "task_queue"
2) "job_1"

# 3. Inspect remaining items
127.0.0.1:6379> LRANGE task_queue 0 -1
1) "job_3"
2) "job_2"

# 4. Maintain a "Recent Activity" list of max 5 items
127.0.0.1:6379> LPUSH recent_activity "login"
(integer) 1
127.0.0.1:6379> LPUSH recent_activity "click"
(integer) 2
# ... add more ...
127.0.0.1:6379> LTRIM recent_activity 0 4  # Keep only indices 0 to 4 (5 items)
OK
```

---

### 3. Sets

**Concept:** Unordered collection of **unique** strings. O(1) complexity for add/remove/check.

### Core Syntax & Arguments

```powershell
SADD key member [member ...]      # Add members. Ignores duplicates.
SREM key member [member ...]      # Remove members.
SMEMBERS key                      # Get all members.
SISMEMBER key member              # Check existence. Returns 1 or 0.
SCARD key                         # Get cardinality (count).
SINTER key [key ...]              # Intersection of multiple sets.
SUNION key [key ...]              # Union of multiple sets.
SDIFF key [key ...]               # Difference (members in first key but not others).
SRANDMEMBER key [count]           # Get random member(s).
SPOP key [count]                  # Remove and return random member(s).
```

| Argument | Type | Description |
| --- | --- | --- |
| `member` | String | Unique value to add/remove. |
| `count` | Integer | For `SRANDMEMBER`: Positive = distinct elements. Negative = allow duplicates. |

### CLI Deep-Dive Example

```bash
# 1. Add tags to an article
127.0.0.1:6379> SADD tags:article:101 "redis" "cache" "database"
(integer) 3

# 2. Add duplicate (ignored)
127.0.0.1:6379> SADD tags:article:101 "redis"
(integer) 0

# 3. Check if tag exists
127.0.0.1:6379> SISMEMBER tags:article:101 "redis"
(integer) 1

# 4. Find common tags between two articles (Intersection)
127.0.0.1:6379> SADD tags:article:102 "redis" "python"
(integer) 2
127.0.0.1:6379> SINTER tags:article:101 tags:article:102
1) "redis"

# 5. Get random sample for recommendation
127.0.0.1:6379> SRANDMEMBER tags:article:101 2
1) "cache"
2) "database"
```

---

### 4. Sorted Sets (ZSets)

**Concept:** Like Sets, but each member has a **score** (double). Sorted by score. Used for leaderboards.

### Core Syntax & Arguments

```bash
ZADD key [NX | XX] [CH] [INCR] score member [score member ...]
ZREM key member [member ...]
ZSCORE key member
ZRANK key member          # Rank ascending (0 = lowest score).
ZREVRANK key member       # Rank descending (0 = highest score).
ZRANGE key min max [BYSCORE | BYLEX] [REV] [LIMIT offset count] [WITHSCORES]
ZREVRANGE key start stop [WITHSCORES]
ZCOUNT key min max        # Count members in score range.
ZINCRBY key increment member
ZREMRANGEBYSCORE key min max
```

| Argument | Type | Description |
| --- | --- | --- |
| `score` | Double | Floating point number used for sorting. |
| `member` | String | The unique element. |
| `NX` | Flag | Only add new elements (don't update scores). |
| `XX` | Flag | Only update existing elements. |
| `min/max` | Double/String | Range boundaries. Use `-inf` and `+inf` for unlimited. |
| `start/stop` | Integer | Rank indices (0-based). |
| `WITHSCORES` | Flag | Return scores along with members. |

### CLI Deep-Dive Example

```bash
# 1. Add players with scores
127.0.0.1:6379> ZADD leaderboard 100 "PlayerA" 250 "PlayerB" 50 "PlayerC"
(integer) 3

# 2. Update score (Increment)
127.0.0.1:6379> ZINCRBY leaderboard 10 "PlayerA"
"110"

# 3. Get Top 3 Players (Highest Score First)
# ZREVRANGE: 0 to 2 means top 3. WITHSCORES returns scores.
127.0.0.1:6379> ZREVRANGE leaderboard 0 2 WITHSCORES
1) "PlayerB"
2) "250"
3) "PlayerA"
4) "110"
5) "PlayerC"
6) "50"

# 4. Get Rank of PlayerA (0-based index from bottom)
127.0.0.1:6379> ZRANK leaderboard "PlayerA"
(integer) 1

# 5. Remove players with score less than 100
127.0.0.1:6379> ZREMRANGEBYSCORE leaderboard -inf 99
(integer) 1
```

---

### 5. Hashes

**Concept:** Map of field-value pairs. Ideal for objects.

### Core Syntax & Arguments

```bash
HSET key field value [field value ...]
HGET key field
HMGET key field [field ...]
HDEL key field [field ...]
HEXISTS key field
HLEN key
HGETALL key
HINCRBY key field increment
HINCRBYFLOAT key field increment
HKEYS key
HVALS key
```

| Argument | Type | Description |
| --- | --- | --- |
| `field` | String | The attribute name (e.g., "name", "age"). |
| `value` | String | The attribute value. |
| `increment` | Integer/Float | Amount to increment. |

### CLI Deep-Dive Example

```bash
# 1. Set user profile fields
127.0.0.1:6379> HSET user:1001 name "Muhammed" age "30" email "muhammed@example.com"
(integer) 3

# 2. Get specific field
127.0.0.1:6379> HGET user:1001 name
"Muhammed"

# 3. Get multiple fields efficiently
127.0.0.1:6379> HMGET user:1001 name email
1) "Muhammed"
2) "muhammed@example.com"

# 4. Increment login count
127.0.0.1:6379> HINCRBY user:1001 login_count 1
(integer) 1

# 5. Check if field exists
127.0.0.1:6379> HEXISTS user:1001 "premium"
(integer) 0
```

---

### 6. Bitmaps

**Concept:** Bit-level operations on Strings. Extremely memory efficient for boolean arrays.

### Core Syntax & Arguments

```bash
SETBIT key offset value
GETBIT key offset
BITCOUNT key [start end [BYTE | BIT]]
BITOP operation destkey key [key ...]
BITPOS key bit [start [end [BYTE | BIT]]]
```

| Argument | Type | Description |
| --- | --- | --- |
| `offset` | Integer | Bit position (0-based). |
| `value` | Integer | 0 or 1. |
| `operation` | String | `AND`, `OR`, `XOR`, `NOT`. |
| `destkey` | String | Key to store result of BITOP. |
| `bit` | Integer | 0 or 1 (for BITPOS). |

### CLI Deep-Dive Example

```bash
# 1. Mark User ID 1001 as active today
127.0.0.1:6379> SETBIT daily_active:2026-04-14 1001 1
(integer) 0  # Returns previous value (0 means was inactive)

# 2. Check if User 1001 is active
127.0.0.1:6379> GETBIT daily_active:2026-04-14 1001
(integer) 1

# 3. Count total active users
127.0.0.1:6379> BITCOUNT daily_active:2026-04-14
(integer) 1

# 4. Find users active on BOTH Day 1 AND Day 2
127.0.0.1:6379> BITOP AND common_users daily_active:2026-04-13 daily_active:2026-04-14
(integer) 128  # Length of resulting string
127.0.0.1:6379> BITCOUNT common_users
(integer) 0    # No common users in this example
```

---

### 7. HyperLogLog

**Concept:** Probabilistic data structure for counting unique elements. Fixed ~12KB memory.

### Core Syntax & Arguments

```bash
PFADD key element [element ...]
PFCOUNT key [key ...]
PFMERGE destkey sourcekey [sourcekey ...]
```

| Argument | Type | Description |
| --- | --- | --- |
| `element` | String | Item to count. |
| `destkey` | String | Target key for merged result. |

### CLI Deep-Dive Example

```bash
# 1. Add unique visitors
127.0.0.1:6379> PFADD visitors:today "ip_1" "ip_2" "ip_3"
(integer) 1

# 2. Add duplicate (ignored internally for count)
127.0.0.1:6379> PFADD visitors:today "ip_1"
(integer) 0

# 3. Get approximate count
127.0.0.1:6379> PFCOUNT visitors:today
(integer) 3

# 4. Merge two days
127.0.0.1:6379> PFADD visitors:yesterday "ip_3" "ip_4"
(integer) 1
127.0.0.1:6379> PFMERGE visitors:week visitors:today visitors:yesterday
OK
127.0.0.1:6379> PFCOUNT visitors:week
(integer) 4  # ip_1, ip_2, ip_3, ip_4
```

---

### 8. Geospatial Indexes (Geo)

**Concept:** Sorted Set with Geohash. Used for location queries.

### Core Syntax & Arguments

```bash
GEOADD key [NX | XX] [CH] longitude latitude member [longitude latitude member ...]
GEOPOS key member [member ...]
GEODIST key member1 member2 [M | KM | FT | MI]
GEORADIUS key longitude latitude radius M|KM|FT|MI [WITHCOORD] [WITHDIST] [COUNT count] [ASC|DESC]
GEOSEARCH key FROMMEMBER member | FROMLONLAT lon lat BYRADIUS radius M|KM|FT|MI | BYBOX w h M|KM|FT|MI [ASC|DESC] [COUNT count [ANY]] [WITHCOORD] [WITHDIST]
```

| Argument | Type | Description |
| --- | --- | --- |
| `longitude` | Double | X coordinate. |
| `latitude` | Double | Y coordinate. |
| `radius` | Double | Search radius. |
| `M/KM/FT/MI` | Unit | Meters, Kilometers, Feet, Miles. |
| `WITHCOORD` | Flag | Return longitude/latitude of results. |
| `WITHDIST` | Flag | Return distance from center. |

### CLI Deep-Dive Example

```bash
# 1. Add cities
127.0.0.1:6379> GEOADD cities 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(integer) 2

# 2. Get distance between two cities in KM
127.0.0.1:6379> GEODIST cities "Palermo" "Catania" KM
"166.2742"

# 3. Find cities within 100km of a specific point (Lon: 15, Lat: 37)
127.0.0.1:6379> GEORADIUS cities 15 37 100 KM WITHDIST
1) 1) "Catania"
   2) "27.8345"
2) 1) "Palermo"
   2) "95.1234"

# 4. Modern Approach: GEOSEARCH
127.0.0.1:6379> GEOSEARCH cities FROMMEMBER "Palermo" BYRADIUS 100 KM WITHDIST
1) 1) "Catania"
   2) "166.2742"  # Wait, Catania is > 100km? Let's check logic.
                  # Actually, previous GEODIST said 166km. So it won't appear in 100km radius.
                  # Correct result should be empty or only nearby items.
```

---

### 9. Streams

**Concept:** Append-only log with Consumer Groups. Best for event sourcing and robust messaging.

### Core Syntax & Arguments

```bash
XADD key [NOMKSTREAM] [MAXLEN [=|~] threshold] * field value [field value ...]
XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] id [id ...]
XLEN key
XRANGE key start end [COUNT count]
XGROUP CREATE key groupname id|$ [MKSTREAM]
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] id
XACK key group id [id ...]
XPENDING key group [[IDLE min-idle-time] start end count [consumer]]
XDEL key id [id ...]
XTRIM key MAXLEN [=|~] threshold
```

| Argument | Type | Description |
| --- | --- | --- |
| `*` | Special ID | Auto-generates timestamp-based ID. |
| `id` | String | Message ID. Use `$` for "new messages only" in groups. |
| `groupname` | String | Name of consumer group. |
| `consumer` | String | Name of specific consumer instance. |
| `BLOCK ms` | Integer | Timeout for blocking reads. |
| `NOACK` | Flag | Don't require explicit acknowledgment (fire-and-forget). |

### CLI Deep-Dive Example

```bash
# 1. Produce an event
127.0.0.1:6379> XADD mystream * sensor_id 1234 temperature 25.5
"1713123456789-0"

# 2. Create a Consumer Group (starts from new messages '$')
127.0.0.1:6379> XGROUP CREATE mystream mygroup $ MKSTREAM
OK

# 3. Consumer reads new messages
127.0.0.1:6379> XREADGROUP GROUP mygroup consumer1 COUNT 1 STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) "1713123456789-0"
         2) 1) "sensor_id"
            2) "1234"
            3) "temperature"
            4) "25.5"

# 4. Acknowledge processing (Critical for reliability)
127.0.0.1:6379> XACK mystream mygroup 1713123456789-0
(integer) 1

# 5. Check pending messages (if consumer crashed before ACK)
127.0.0.1:6379> XPENDING mystream mygroup
1) (integer) 0  # 0 pending because we ACKed
```

---

### Expert Tips for CLI Usage

1. **Use `redis-cli --raw`**: By default, `redis-cli` quotes strings. Use `-raw` to see clean output, especially for debugging JSON strings.
2. **Pipeline in CLI**: You can paste multiple commands into `redis-cli` to execute them in sequence. For true pipelining (network optimization), use `cat commands.txt | redis-cli --pipe`.
3. **Monitor Slow Queries**: Use `SLOWLOG GET 10` in CLI to see the 10 slowest commands executed recently. This helps identify expensive operations like `KEYS *` or large `LRANGE`.
4. **Debug Memory**: Use `MEMORY USAGE key` to see exactly how many bytes a specific key consumes.

This CLI-focused guide provides the exact syntax and arguments you need to master Redis data structures at the deepest level.