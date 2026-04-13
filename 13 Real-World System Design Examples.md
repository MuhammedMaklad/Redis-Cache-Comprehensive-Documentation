# 13. Real-World System Design Examples

Let's apply our knowledge to common system design challenges, demonstrating how Redis can be a powerful component.

### Designing a Caching Layer for a High-Traffic API

**Scenario**: A REST API serves product information, and the underlying database is becoming a bottleneck due to high read traffic.

**Solution using Redis (Cache-Aside Pattern)**:

1. **Cache Layer**: Introduce Redis as a cache between the API and the database.
2. **Read Flow**: When a request for product `ID` comes in:
    - API checks Redis for `product:ID`.
    - If found (cache hit), return data from Redis.
    - If not found (cache miss), query the database for `product:ID`.
    - Store the retrieved product data in Redis with an appropriate TTL (e.g., 1 hour). Return data to client.
3. **Write Flow**: When a product is updated:
    - Update the product in the primary database.
    - Invalidate (delete) `product:ID` from Redis to ensure consistency.
4. **Eviction Policy**: Configure Redis with `maxmemory` and an eviction policy like `allkeys-lru` to automatically remove least recently used items when memory is full.
5. **High Availability**: Deploy Redis in a master-replica setup with Sentinel for automatic failover.

**Benefits**: Reduces database load, improves API response times, handles traffic spikes more gracefully.

### Rate Limiter for Login System

**Scenario**: Prevent brute-force attacks on a login endpoint by limiting the number of login attempts per IP address or username within a time window.

**Solution using Redis (Fixed Window Counter)**:

1. **Key Design**: Use a Redis key like `rate_limit:login:<IP_ADDRESS>` or `rate_limit:login:<USERNAME>`.
2. **Logic**: For each login attempt:
    - Increment the counter for the relevant key using `INCR`.
    - Set an `EXPIRE` on the key for the duration of the rate limit window (e.g., 5 minutes). If the key already has an expiry, `INCR` will not reset it.
    - Check the current value of the counter. If it exceeds the allowed limit (e.g., 5 attempts), reject the login attempt.
3. **Atomicity**: Use Redis `PIPELINE` to ensure `INCR` and `EXPIRE` are executed atomically to prevent race conditions.

**Benefits**: Fast, accurate, and distributed rate limiting. Prevents attackers from overwhelming the authentication service.

### Real-time Chat System Using Redis

**Scenario**: Build a real-time chat application where users can join channels and exchange messages.

**Solution using Redis (Pub/Sub and Streams)**:

1. **Channel Messaging (Pub/Sub)**:
    - When a user sends a message to a chat channel (e.g., `general_chat`), the application `PUBLISH`es the message to a Redis Pub/Sub channel (e.g., `chat:general`).
    - All connected clients subscribed to `chat:general` will receive the message in real-time.
    - **Limitation**: Pub/Sub is fire-and-forget. If a user is offline, they miss messages.
2. **Message History (Streams)**:
    - For message persistence and history, use Redis Streams. Every message sent to a chat channel is also `XADD`ed to a corresponding Redis Stream (e.g., `chat:general:history`).
    - When a user connects, they can `XRANGE` the stream to retrieve recent message history.
    - For persistent consumption by microservices (e.g., for analytics or archiving), use Redis Stream Consumer Groups.

**Benefits**: Provides both real-time communication and message persistence. Scales well for many users and channels.

### Distributed Locking in Microservices

**Scenario**: In a microservices architecture, multiple instances of a service might try to update the same shared resource (e.g., decrementing inventory, processing an order) concurrently, leading to race conditions.

**Solution using Redis (SET NX PX)**:

1. **Lock Acquisition**: When a service instance needs to access a critical section:
    - It attempts to acquire a lock using `SET resource_lock:<ID> <UNIQUE_VALUE> NX PX <TTL_MILLISECONDS>`.
    - `NX` ensures the lock is only set if it doesn't already exist.
    - `PX` sets an expiration time to prevent deadlocks if the service instance crashes.
    - If `SET` returns `OK`, the lock is acquired.
2. **Critical Section**: The service performs its operation on the shared resource.
3. **Lock Release**: After completing the operation, the service releases the lock. This must be done atomically using a Lua script to ensure that only the owner of the lock can delete it (preventing a service from deleting a lock that expired and was re-acquired by another service).

**Benefits**: Ensures mutual exclusion for shared resources in a distributed environment, preventing data corruption and race conditions.