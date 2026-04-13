# 7. Redis As Rate Limiting

**Description**: **Rate limiting** ***controls the number of requests a user or service can make to an API within a given time window***. Redis is excellent for this due to its fast atomic operations.

**Mechanism**: Common approaches include:

- **Fixed Window Counter**: Increment a counter for a given time window. If the counter exceeds the limit, reject the request.
- **Sliding Window Log**: Store timestamps of each request in a Redis List or Sorted Set. When a new request comes, remove old timestamps and check the count.

**Real-World Scenarios**:

- **API Protection**: Preventing abuse and ensuring fair usage of public APIs.
- **Login Brute-Force Protection**: Limiting login attempts from a single IP address.

**Code Examples (Python - Fixed Window Counter)**:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, db=0)

def is_rate_limited(user_id, limit=5, window_seconds=60):
    """
    Checks if a user has exceeded their rate limit using a Fixed Window Counter.
    
    Args:
        user_id (str): Unique identifier for the user.
        limit (int): Max allowed requests per window.
        window_seconds (int): Duration of the time window in seconds.
        
    Returns:
        bool: True if rate limited (blocked), False if allowed.
    """
    
    # 1. Define the Redis Key
    # We namespace the key with 'rate_limit:' to avoid collisions with other data.
    key = f"rate_limit:{user_id}"
    
    # 2. Initialize a Pipeline
    # A pipeline batches multiple commands into a single network round-trip.
    # This reduces latency and ensures the commands are executed sequentially.
    pipe = r.pipeline()
    
    # 3. Increment the Counter
    # INCR creates the key if it doesn't exist (starting at 0) and increments it by 1.
    # This represents the current request being made.
    pipe.incr(key)
    
    # 4. Set Expiration (TTL)
    # EXPIRE sets the time-to-live for the key. 
    # IMPORTANT: This only sets the expiration if the key is NEW or already exists.
    # In a pipeline, this runs immediately after INCR.
    pipe.expire(key, window_seconds)
    
    # 5. Execute the Pipeline
    # Sends both commands to Redis at once.
    # Returns a list of results: [result_of_incr, result_of_expire]
    # result_of_incr: The new count (integer).
    # result_of_expire: True/False (whether TTL was set).
    current_count, _ = pipe.execute()
    
    # 6. Check Limit
    # If the count exceeds the limit, return True (Blocked).
    # Otherwise, return False (Allowed).
    return current_count > limit

# Example usage:
user = "user:alice"
for i in range(10):
    if is_rate_limited(user):
        print(f"Request {i+1} for {user}: Rate limited!")
    else:
        print(f"Request {i+1} for {user}: OK")
    time.sleep(0.5)

# Output will show first 5 as OK, then Rate limited.
```

```python
import time
import redis

r = redis.Redis(host='localhost', port=6379, db=0)

def is_rate_limited_sliding(user_id, limit=5, window_seconds=60):
    """
    Rate limiting using a Sliding Window Log with Redis Sorted Sets.
    More accurate than Fixed Window, but uses more memory.
    """
    key = f"rate_limit_sliding:{user_id}"
    now = time.time()
    window_start = now - window_seconds
    
    # Use a pipeline for atomicity
    pipe = r.pipeline()
    
    # 1. Remove old entries outside the current window
    # ZREMRANGEBYSCORE removes members with scores (timestamps) < window_start
    pipe.zremrangebyscore(key, 0, window_start)
    
    # 2. Add the current request with timestamp as score
    # ZADD adds the member (unique ID like timestamp+random) with score=now
    pipe.zadd(key, {f"{now}:{id(user_id)}": now})
    
    # 3. Set expiry on the key itself to clean up empty sets
    pipe.expire(key, window_seconds)
    
    # 4. Count the number of requests in the current window
    pipe.zcard(key)
    
    # Execute all commands
    _, _, _, current_count = pipe.execute()
    
    # If count exceeds limit, remove the current request (rollback) and block
    if current_count > limit:
        r.zrem(key, f"{now}:{id(user_id)}")  # Optional: Remove the request that triggered the limit
        return True  # Blocked
    
    return False  # Allowed
```