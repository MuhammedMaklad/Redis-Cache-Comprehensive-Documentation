# 7. Redis As Distributed Locks (SET NX, Redlock)

**Description**: In distributed systems, you often ***need to ensure that only one process at a time can access a shared resource or execute a critical section of code***. ***Redis can be used to implement distributed locks.***

**Mechanism**: The simplest form uses `SET key value NX PX milliseconds`.

 `NX` ensures the key is set only if it doesn't already exist.

 `PX` sets an expiration time, preventing deadlocks if a client crashes.

**Redlock**: For higher guarantees in a distributed Redis environment (e.g., Redis Cluster), the Redlock algorithm is used. It involves acquiring locks on a majority of independent Redis master instances to ensure robustness against single-point failures.

**Real-World Scenarios**:

- **Preventing Double Submissions**: Ensuring a user can't submit the same form twice.
- **Resource Access Control**: Allowing only one microservice instance to update a specific database record.
- **Task Scheduling**: Ensuring a scheduled task runs only once across multiple worker nodes.

**Code Examples (Python - simplified `SET NX` lock)**:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, db=0)

def acquire_lock(lock_name, acquire_timeout=10, expire_time=5):
    identifier = str(time.time()) # Unique identifier for the lock owner
    end_time = time.time() + acquire_timeout
    while time.time() < end_time:
        if r.set(lock_name, identifier, nx=True, ex=expire_time):
            return identifier
        time.sleep(0.001)
    return False

def release_lock(lock_name, identifier):
    # Use a Lua script for atomic check-and-delete
    # This prevents a race condition where a client deletes a lock set by another client
    lua_script = """
    if redis.call("get",KEYS[1]) == ARGV[1] then
        return redis.call("del",KEYS[1])
    else
        return 0
    end
    """
    return r.eval(lua_script, 1, lock_name, identifier)

# Example usage:
lock_id = acquire_lock('my_resource_lock')
if lock_id:
    print(f"Acquired lock with ID: {lock_id}")
    try:
        # Critical section: access shared resource
        print("Performing critical operation...")
        time.sleep(2) # Simulate work
    finally:
        release_lock('my_resource_lock', lock_id)
        print("Released lock.")
else:
    print("Could not acquire lock.")
```