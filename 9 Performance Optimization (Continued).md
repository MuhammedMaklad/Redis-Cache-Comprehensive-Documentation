# 9. Performance Optimization (Continued)

### Pipelining

**Description**: Pipelining is a technique that allows a client to send multiple commands to the Redis server without waiting for the replies to the previous commands. Redis then processes these commands in a batch and sends all the replies back to the client in a single response. This significantly reduces the **Round Trip Time (RTT)** overhead, especially over high-latency networks.

**Real Performance Scenarios**:

- **Batching writes**: When inserting many items into a list or hash, or setting multiple keys.
- **Atomic operations**: While not a transaction, pipelining ensures that commands are executed sequentially on the server without interleaving from other clients.

**Example (Python)**:

```python
import redis
import time

r = redis.Redis(host="localhost", port=6379, db=0)

start_time = time.time()
for i in range(1000):
    r.set(f"key:{i}", f"value:{i}")
end_time = time.time()
print(f"Without pipelining: {end_time - start_time:.4f} seconds")

start_time = time.time()
pipeline = r.pipeline()
for i in range(1000):
    pipeline.set(f"key:{i}", f"value:{i}")
pipeline.execute()
end_time = time.time()
print(f"With pipelining: {end_time - start_time:.4f} seconds")
```

### Batching

**Description**: Batching is a broader concept than pipelining. It refers to grouping multiple operations together to reduce overhead. Pipelining is a form of batching specific to network communication with Redis. Other forms of batching might involve using multi-key commands (e.g., `MSET`, `MGET`) or Lua scripting to execute a block of commands atomically on the server.

**Real Performance Scenarios**:

- **Loading initial data**: Use `MSET` or a pipeline of `SET` commands.
- **Retrieving multiple values**: Use `MGET` instead of multiple `GET` commands.
- **Complex atomic updates**: Use Lua scripts to perform multiple operations that need to be atomic and efficient, reducing network round trips and ensuring server-side atomicity.