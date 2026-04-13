# 6. Redis in Backend Systems

Integrating Redis into backend applications is straightforward across various programming languages. The key is to understand connection management, basic operations, and best practices for each ecosystem.

### Node.js (ioredis / redis client)

Node.js applications often use `ioredis` or the official `redis` client for interacting with Redis. `ioredis` is a robust, high-performance client that supports Redis Cluster, Sentinel, and pipelining.

**Connection Setup (ioredis)**:

```jsx
// Import the ioredis library, a robust Redis client for Node.js
const Redis = require('ioredis');

/**
 * Create a new Redis instance.
 * This connects to a single standalone Redis server.
 */
const redis = new Redis({
  port: 6379,          // The port number where Redis is listening (default is 6379)
  host: '127.0.0.1',   // The IP address or hostname of the Redis server (localhost)
  password: 'your_redis_password', // Authentication password. Remove this line if no password is set.
  db: 0,               // The database index to select after connecting (0-15 by default). Defaults to 0.
  
  // Optional common configurations you might add:
  // maxRetriesPerRequest: 3, // How many times to retry a command before failing
  // retryStrategy: (times) => Math.min(times * 50, 2000), // Custom retry logic
});

/**
 * Event Listener: 'connect'
 * Triggered when the connection to the Redis server is successfully established.
 */
redis.on('connect', () => {
  console.log('Connected to Redis!');
});

/**
 * Event Listener: 'error'
 * Triggered if there is an error with the connection (e.g., wrong password, server down).
 * It is crucial to handle this to prevent your Node.js app from crashing.
 */
redis.on('error', (err) => {
  console.error('Redis Client Error', err);
});

// -----------------------------------------------------------------------------
// ALTERNATIVE CONNECTION METHODS (Commented Out)
// -----------------------------------------------------------------------------

/**
 * OPTION 2: Connect to a Redis Cluster
 * Use this if you are running a sharded Redis Cluster for high availability/scalability.
 * You provide a list of nodes; ioredis will discover the rest of the cluster topology.
 */
// const cluster = new Redis.Cluster([
//   { port: 6379, host: '127.0.0.1' }, // Seed node 1
//   { port: 6380, host: '127.0.0.1' }, // Seed node 2
// ], {
//   // Optional cluster-specific options:
//   // redisOptions: { password: 'your_cluster_password' } // If all nodes share the same password
// });

// cluster.on('connect', () => console.log('Connected to Redis Cluster!'));
// cluster.on('error', (err) => console.error('Cluster Error', err));

/**
 * OPTION 3: Connect via Redis Sentinel
 * Use this for High Availability (HA) setups where Sentinel monitors master/slave instances.
 * ioredis will automatically find the current master based on the sentinel configuration.
 */
// const sentinel = new Redis({
//   sentinels: [
//     { host: '127.0.0.1', port: 26379 }, // Sentinel instance 1
//     { host: '127.0.0.1', port: 26380 }, // Sentinel instance 2
//   ],
//   name: 'mymaster', // The name of the master group as defined in sentinel.conf
//   
//   // Optional:
//   // password: 'your_redis_password', // Password for the actual Redis data nodes
//   // sentinelPassword: 'your_sentinel_password' // Password for the Sentinel nodes themselves
// });

// sentinel.on('connect', () => console.log('Connected to Redis Sentinel!'));
// sentinel.on('error', (err) => console.error('Sentinel Error', err));
```

**Basic Operations (ioredis)**:

```jsx
async function performRedisOperations() {
  try {
    // Strings
    await redis.set('mykey', 'Hello from Node.js');
    const value = await redis.get('mykey');
    console.log('GET mykey:', value); // Hello from Node.js

    // Hashes
    await redis.hset('user:1', 'name', 'Alice', 'email', 'alice@example.com');
    const user = await redis.hgetall('user:1');
    console.log('HGETALL user:1:', user); // { name: 'Alice', email: 'alice@example.com' }

    // Lists
    await redis.lpush('mylist', 'item1', 'item2');
    const listItems = await redis.lrange('mylist', 0, -1);
    console.log('LRANGE mylist:', listItems); // [ 'item2', 'item1' ]

    // Sets
    await redis.sadd('myset', 'member1', 'member2');
    const isMember = await redis.sismember('myset', 'member1');
    console.log('SISMEMBER myset member1:', isMember); // 1

    // Sorted Sets
    await redis.zadd('myzset', 1, 'one', 2, 'two');
    const zsetItems = await redis.zrange('myzset', 0, -1, 'WITHSCORES');
    console.log('ZRANGE myzset:', zsetItems); // [ 'one', '1', 'two', '2' ]

    // Pipelining (send multiple commands in one round trip)
    const pipelineResult = await redis.pipeline()
      .set('key1', 'value1')
      .get('key1')
      .incr('counter')
      .exec();
    console.log('Pipeline Result:', pipelineResult);
    // [[null, 'OK'], [null, 'value1'], [null, 1]]

  } catch (error) {
    console.error('Redis operation failed:', error);
  } finally {
    redis.disconnect();
  }
}

performRedisOperations();
```

**Best Practices (Node.js)**:

- **Connection Pooling**: `ioredis` manages connections efficiently. Reuse a single `Redis` instance throughout your application rather than creating a new one for each operation.
- **Error Handling**: Always attach error listeners to your Redis client to catch connection issues and other errors.
- **Pipelining**: For multiple commands that don't depend on each other's results, use pipelining to reduce network round-trip times and improve throughput.
- **Async/Await**: Leverage `async/await` for cleaner asynchronous Redis operations.
- **Configuration Management**: Store Redis connection details in environment variables or a secure configuration service.

### Python (redis-py)

`redis-py` is the most popular and feature-rich Python client for Redis. It provides a Pythonic interface to all Redis commands.

**Connection Setup (redis-py)**:

```python
import redis

# Connect to a single Redis instance
try:
    r = redis.Redis(host='localhost', port=6379, db=0, password='your_redis_password')
    r.ping() # Test connection
    print("Connected to Redis!")
except redis.exceptions.ConnectionError as e:
    print(f"Could not connect to Redis: {e}")

# Connect to Redis Sentinel
# from redis.sentinel import Sentinel
# sentinel = Sentinel([('localhost', 26379), ('localhost', 26380)], socket_timeout=0.1)
# master = sentinel.master_for('mymaster', socket_timeout=0.1)
# slave = sentinel.slave_for('mymaster', socket_timeout=0.1)
# print("Connected to Redis Sentinel!")

# Connect to Redis Cluster
# from redis.cluster import RedisCluster
# cluster = RedisCluster(host='localhost', port=7000, decode_responses=True)
# cluster.ping()
# print("Connected to Redis Cluster!")
```

**Basic Operations (redis-py)**:

```python
# Strings
r.set('mykey', 'Hello from Python')
value = r.get('mykey')
print(f"GET mykey: {value.decode('utf-8')}") # Decode bytes to string

# Hashes
r.hset('user:2', mapping={'name': 'Bob', 'email': 'bob@example.com'})
user = r.hgetall('user:2')
print(f"HGETALL user:2: { {k.decode('utf-8'): v.decode('utf-8') for k, v in user.items()} }")

# Lists
r.lpush('mylist_py', 'item_py1', 'item_py2')
list_items = r.lrange('mylist_py', 0, -1)
print(f"LRANGE mylist_py: {[item.decode('utf-8') for item in list_items]}")

# Sets
r.sadd('myset_py', 'member_py1', 'member_py2')
is_member = r.sismember('myset_py', 'member_py1')
print(f"SISMEMBER myset_py member_py1: {is_member}")

# Sorted Sets
r.zadd('myzset_py', {'one': 1, 'two': 2})
zset_items = r.zrange('myzset_py', 0, -1, withscores=True)
print(f"ZRANGE myzset_py: {zset_items}")

# Pipelining
pipeline = r.pipeline()
pipeline.set('key_py1', 'value_py1')
pipeline.get('key_py1')
pipeline.incr('counter_py')
result = pipeline.execute()
print(f"Pipeline Result: {result}")
```

**Best Practices (Python)**:

- **Connection Management**: `redis-py` handles connection pooling automatically. Create a single `Redis` client instance and reuse it.
- **Decode Responses**: Use `decode_responses=True` when initializing the client if you primarily work with strings to avoid manually decoding bytes.
- **Pipelining**: Use `pipeline()` for batching commands to reduce latency.
- **Transactions (WATCH/MULTI/EXEC)**: For atomic operations involving multiple keys, use Redis transactions with `WATCH` to ensure data consistency.
- **Error Handling**: Implement robust `try-except` blocks to handle `redis.exceptions.ConnectionError` and other potential issues.

### .NET (StackExchange.Redis)

`StackExchange.Redis` is a high-performance, feature-rich Redis client for .NET applications. It's widely used and supports advanced features like multiplexing, pipelining, and pub/sub.

**Connection Setup (StackExchange.Redis)**:

```csharp
using StackExchange.Redis;
using System;
using System.Threading.Tasks;

public class RedisService
{
    private static ConnectionMultiplexer _redis;
    private static IDatabase _db;

    public static async Task InitializeAsync()
    {
        if (_redis == null || !_redis.IsConnected)
        {
            try
            {
                // ConfigurationOptions allows for detailed setup
                // e.g., "localhost:6379,password=your_redis_password,syncTimeout=5000"
                _redis = await ConnectionMultiplexer.ConnectAsync("localhost:6379");
                _db = _redis.GetDatabase();
                Console.WriteLine("Connected to Redis!");
            }
            catch (RedisConnectionException ex)
            {
                Console.WriteLine($"Could not connect to Redis: {ex.Message}");
            }
        }
    }

    public static IDatabase GetDatabase() => _db;

    public static void CloseConnection()
    {
        _redis?.Close();
    }
}

// Usage in your application:
// await RedisService.InitializeAsync();
// IDatabase db = RedisService.GetDatabase();
```

**Basic Operations (StackExchange.Redis)**:

```csharp
// Example usage after initialization
public static async Task PerformRedisOperations(IDatabase db)
{
    // Strings
    await db.StringSetAsync("mykey_cs", "Hello from C#");
    string value = await db.StringGetAsync("mykey_cs");
    Console.WriteLine($"GET mykey_cs: {value}");

    // Hashes
    await db.HashSetAsync("user:3", new HashEntry[]
    {
        new HashEntry("name", "Charlie"),
        new HashEntry("email", "charlie@example.com")
    });
    HashEntry[] user = await db.HashGetAllAsync("user:3");
    foreach (var entry in user)
    {
        Console.WriteLine($"  {entry.Name}: {entry.Value}");
    }

    // Lists
    await db.ListLeftPushAsync("mylist_cs", "item_cs1", "item_cs2");
    RedisValue[] listItems = await db.ListRangeAsync("mylist_cs", 0, -1);
    Console.WriteLine($"LRANGE mylist_cs: {string.Join(", ", listItems)}");

    // Pipelining (Batching)
    var batch = db.CreateBatch();
    var task1 = batch.StringSetAsync("key_cs1", "value_cs1");
    var task2 = batch.StringGetAsync("key_cs1");
    var task3 = batch.StringIncrementAsync("counter_cs");
    batch.Execute(); // Send all commands to Redis

    await Task.WhenAll(task1, task2, task3);
    Console.WriteLine($"Batch Result 1: {await task1}");
    Console.WriteLine($"Batch Result 2: {await task2}");
    Console.WriteLine($"Batch Result 3: {await task3}");
}

// Call it:
// await PerformRedisOperations(db);
```

**Best Practices (.NET)**:

- **ConnectionMultiplexer**: The `ConnectionMultiplexer` is designed to be shared and reused throughout your application. Creating a new one for each operation is expensive.
- **Asynchronous Operations**: `StackExchange.Redis` is heavily asynchronous. Use `async/await` for non-blocking I/O.
- **Batching/Pipelining**: Use `CreateBatch()` for pipelining multiple commands to reduce network overhead.
- **Error Handling and Resilience**: Implement retry logic and circuit breakers for transient network issues. Monitor connection health.
- **Configuration**: Use `ConfigurationOptions` for advanced connection settings, including password, SSL, and timeouts.