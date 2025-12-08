# Memcached NoSQL Architecture

## Overview

Memcached is a high-performance, distributed memory caching system designed to speed up dynamic web applications by reducing database load. It's a simple key-value store that keeps data entirely in RAM.

## Core Characteristics

- **In-Memory**: All data stored in RAM (volatile)
- **Key-Value Store**: Simple data model (string keys, binary values)
- **Distributed**: Data spread across multiple servers
- **No Persistence**: Data lost on restart
- **LRU Eviction**: Least Recently Used items removed when memory full
- **Simple Protocol**: Text-based protocol over TCP/UDP

## Architecture

### Basic Structure
```
Client Application
       ↓
Memcached Client Library (Consistent Hashing)
       ↓
┌──────────┬──────────┬──────────┐
│ Memcached│ Memcached│ Memcached│
│ Server 1 │ Server 2 │ Server 3 │
│ (RAM)    │ (RAM)    │ (RAM)    │
└──────────┴──────────┴──────────┘
```

### Key Components

#### 1. Client Library
- Implements consistent hashing
- Determines which server stores each key
- Handles connection pooling
- No server-to-server communication

#### 2. Memcached Server
- Stores key-value pairs in memory
- Handles get/set/delete operations
- Manages memory with slab allocator
- Evicts old data when full

#### 3. Slab Allocator
```
Memory divided into slabs (1MB chunks)
Each slab divided into fixed-size chunks
Different slab classes for different sizes

Slab Class 1: 96 bytes
Slab Class 2: 120 bytes
Slab Class 3: 152 bytes
...
Slab Class 42: 1MB
```

## How It Works

### 1. Consistent Hashing
```python
# Client determines server using hash
def get_server(key):
    hash_value = hash(key)
    server_index = hash_value % num_servers
    return servers[server_index]

# Example
key = "user:1234"
server = get_server(key)  # Returns server2
# All operations for this key go to server2
```

### 2. Data Storage
```
Key: "user:1234"
Value: {"name": "John", "email": "john@example.com"}
Expiration: 3600 seconds
Flags: 0

Stored as:
┌─────────────────────────────────┐
│ Key: user:1234                  │
│ Value: (binary data)            │
│ Expiration: timestamp + 3600    │
│ Flags: 0                        │
│ CAS: 12345 (version)            │
└─────────────────────────────────┘
```

### 3. LRU Eviction
```
Memory Full → Evict Least Recently Used Item

Access Pattern:
key1 (accessed 10 min ago)
key2 (accessed 5 min ago)
key3 (accessed 1 min ago)  ← Most recent

When memory full:
Evict key1 (oldest access)
```

## Basic Operations

### Set (Store Data)
```python
import memcache

mc = memcache.Client(['127.0.0.1:11211'])

# Set with expiration (3600 seconds)
mc.set('user:1234', {'name': 'John', 'age': 30}, time=3600)

# Set multiple
mc.set_multi({
    'user:1234': {'name': 'John'},
    'user:5678': {'name': 'Jane'}
}, time=3600)
```

### Get (Retrieve Data)
```python
# Get single key
user = mc.get('user:1234')
print(user)  # {'name': 'John', 'age': 30}

# Get multiple keys
users = mc.get_multi(['user:1234', 'user:5678'])
print(users)  # {'user:1234': {...}, 'user:5678': {...}}
```

### Delete
```python
# Delete key
mc.delete('user:1234')

# Delete multiple
mc.delete_multi(['user:1234', 'user:5678'])
```

### Increment/Decrement
```python
# Set counter
mc.set('page_views', 0)

# Increment
mc.incr('page_views', 1)  # Returns 1
mc.incr('page_views', 5)  # Returns 6

# Decrement
mc.decr('page_views', 2)  # Returns 4
```

### Add (Set if not exists)
```python
# Add only if key doesn't exist
mc.add('user:1234', {'name': 'John'})  # True if added
mc.add('user:1234', {'name': 'Jane'})  # False (already exists)
```

### Replace (Set if exists)
```python
# Replace only if key exists
mc.replace('user:1234', {'name': 'Jane'})  # True if replaced
mc.replace('user:9999', {'name': 'Bob'})   # False (doesn't exist)
```

### CAS (Compare and Swap)
```python
# Atomic update with version checking
value, cas_id = mc.gets('counter')  # Get with CAS ID
new_value = value + 1
success = mc.cas('counter', new_value, cas_id)  # Only updates if CAS matches
```

## Common Use Cases

### 1. Database Query Caching
```python
def get_user(user_id):
    cache_key = f'user:{user_id}'
    
    # Try cache first
    user = mc.get(cache_key)
    if user:
        return user  # Cache hit
    
    # Cache miss - query database
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # Store in cache
    mc.set(cache_key, user, time=3600)
    return user
```

### 2. Session Storage
```python
def save_session(session_id, session_data):
    mc.set(f'session:{session_id}', session_data, time=1800)

def get_session(session_id):
    return mc.get(f'session:{session_id}')

def delete_session(session_id):
    mc.delete(f'session:{session_id}')
```

### 3. Rate Limiting
```python
def check_rate_limit(user_id, limit=100):
    key = f'rate_limit:{user_id}:{current_minute()}'
    
    count = mc.get(key) or 0
    if count >= limit:
        return False  # Rate limit exceeded
    
    mc.incr(key, 1)
    mc.set(key, count + 1, time=60)  # Expire after 1 minute
    return True
```

### 4. Fragment Caching
```python
def render_page(page_id):
    # Cache rendered HTML fragments
    header = mc.get('fragment:header')
    if not header:
        header = render_header()
        mc.set('fragment:header', header, time=3600)
    
    content = mc.get(f'fragment:page:{page_id}')
    if not content:
        content = render_content(page_id)
        mc.set(f'fragment:page:{page_id}', content, time=1800)
    
    return header + content
```

### 5. Leaderboard/Counters
```python
def increment_score(user_id, points):
    key = f'score:{user_id}'
    mc.incr(key, points)

def get_score(user_id):
    return mc.get(f'score:{user_id}') or 0
```

## Distributed Setup

### Multiple Servers
```python
# Client connects to multiple servers
mc = memcache.Client([
    '192.168.1.10:11211',
    '192.168.1.11:11211',
    '192.168.1.12:11211'
])

# Client library automatically distributes keys
mc.set('key1', 'value1')  # Goes to server1
mc.set('key2', 'value2')  # Goes to server2
mc.set('key3', 'value3')  # Goes to server3
```

### Consistent Hashing
```
Without Consistent Hashing:
- Add/remove server → rehash ALL keys
- 3 servers → 4 servers: ~75% keys move

With Consistent Hashing:
- Add/remove server → only K/N keys move
- 3 servers → 4 servers: ~25% keys move
```

### Example: Adding Server
```python
# Before: 3 servers
servers = ['server1:11211', 'server2:11211', 'server3:11211']
mc = memcache.Client(servers)

# After: 4 servers (minimal disruption)
servers = ['server1:11211', 'server2:11211', 'server3:11211', 'server4:11211']
mc = memcache.Client(servers)

# Only ~25% of keys need to be re-cached
```

## Memory Management

### Slab Allocation
```python
# Check stats
stats = mc.get_stats()
for server, data in stats:
    print(f"Server: {server}")
    print(f"Total memory: {data['limit_maxbytes']}")
    print(f"Used memory: {data['bytes']}")
    print(f"Items: {data['curr_items']}")
    print(f"Evictions: {data['evictions']}")
```

### Monitoring Evictions
```python
# High evictions = need more memory
stats = mc.get_stats()[0][1]
evictions = int(stats['evictions'])
items = int(stats['curr_items'])

if evictions > items * 0.1:  # More than 10% eviction rate
    print("Warning: High eviction rate, consider adding memory")
```

## Cache Strategies

### 1. Cache-Aside (Lazy Loading)
```python
def get_data(key):
    # Check cache
    data = mc.get(key)
    if data:
        return data
    
    # Load from database
    data = db.query(key)
    mc.set(key, data, time=3600)
    return data
```

### 2. Write-Through
```python
def save_data(key, data):
    # Write to database
    db.save(key, data)
    
    # Update cache
    mc.set(key, data, time=3600)
```

### 3. Write-Behind (Write-Back)
```python
def save_data(key, data):
    # Write to cache immediately
    mc.set(key, data, time=3600)
    
    # Queue for async database write
    queue.add_task(lambda: db.save(key, data))
```

### 4. Cache Invalidation
```python
def update_user(user_id, new_data):
    # Update database
    db.update(user_id, new_data)
    
    # Invalidate cache
    mc.delete(f'user:{user_id}')
```

## Performance Characteristics

### Speed
```
Operation    | Time
-------------|--------
GET          | ~1ms
SET          | ~1ms
DELETE       | ~1ms
Multi-GET    | ~2ms (100 keys)
```

### Throughput
```
Single Server:
- ~50,000 GET/sec
- ~25,000 SET/sec

3 Server Cluster:
- ~150,000 GET/sec
- ~75,000 SET/sec
```

## Limitations

### 1. No Persistence
```python
# Data lost on restart
mc.set('important_data', value)
# Server restarts
mc.get('important_data')  # Returns None
```

### 2. No Replication
```python
# Server fails → data lost
# No automatic failover
# No data redundancy
```

### 3. Key Size Limit
```python
# Max key size: 250 bytes
key = 'x' * 251
mc.set(key, 'value')  # Error: key too long
```

### 4. Value Size Limit
```python
# Default max value: 1MB
large_value = 'x' * (1024 * 1024 + 1)
mc.set('key', large_value)  # Error: value too large
```

### 5. No Complex Queries
```python
# Can't query by value
# Can't search keys
# Can't filter data
# Only exact key lookups
```

## Memcached vs Redis

| Feature | Memcached | Redis |
|---------|-----------|-------|
| Data Types | String only | String, List, Set, Hash, etc. |
| Persistence | No | Yes (RDB, AOF) |
| Replication | No | Yes (master-slave) |
| Transactions | No | Yes |
| Pub/Sub | No | Yes |
| Lua Scripts | No | Yes |
| Multi-threading | Yes | No (single-threaded) |
| Memory Efficiency | Better for simple cache | Better for complex data |
| Speed | Slightly faster | Very fast |
| Use Case | Pure caching | Cache + data structure |

## Best Practices

### 1. Key Naming Convention
```python
# Use namespaces
'user:1234'
'session:abc123'
'cache:query:users:active'

# Include version
'user:v2:1234'
```

### 2. Set Appropriate TTL
```python
# Short-lived data
mc.set('session:123', data, time=1800)  # 30 minutes

# Long-lived data
mc.set('config:app', data, time=86400)  # 24 hours

# Frequently changing data
mc.set('stock:price', data, time=60)  # 1 minute
```

### 3. Handle Cache Misses
```python
def get_with_fallback(key, fetch_func, ttl=3600):
    data = mc.get(key)
    if data is None:
        data = fetch_func()
        if data:
            mc.set(key, data, time=ttl)
    return data
```

### 4. Batch Operations
```python
# Use multi-get for multiple keys
keys = [f'user:{i}' for i in range(100)]
users = mc.get_multi(keys)  # Single network call

# Better than
for key in keys:
    user = mc.get(key)  # 100 network calls
```

### 5. Monitor and Alert
```python
def check_health():
    stats = mc.get_stats()[0][1]
    
    # Check hit rate
    hits = int(stats['get_hits'])
    misses = int(stats['get_misses'])
    hit_rate = hits / (hits + misses) if (hits + misses) > 0 else 0
    
    if hit_rate < 0.8:
        alert("Low cache hit rate")
    
    # Check evictions
    if int(stats['evictions']) > 1000:
        alert("High eviction rate")
```

## When to Use Memcached

**Good For:**
- Simple key-value caching
- Session storage
- Database query caching
- Fragment caching
- Rate limiting
- Temporary data storage
- High-throughput scenarios

**Not Good For:**
- Persistent storage
- Complex data structures
- Data that must survive restarts
- Transactions
- Pub/Sub messaging
- Data replication needs

## Conclusion

Memcached is a simple, fast, distributed memory cache ideal for reducing database load and speeding up applications. Its simplicity is both its strength (easy to use, very fast) and limitation (no persistence, no complex features). Use it for pure caching scenarios where data loss on restart is acceptable.
