# NoSQL Redis Internals

## Overview

Redis (Remote Dictionary Server) is an in-memory data structure store used as a database, cache, and message broker. Unlike simple key-value stores, Redis supports rich data structures and provides persistence, replication, and advanced features.

## Core Architecture

### Single-Threaded Event Loop
```
┌─────────────────────────────────┐
│     Client Connections          │
└──────────┬──────────────────────┘
           ↓
┌─────────────────────────────────┐
│   Event Loop (Single Thread)    │
│  - Accept connections           │
│  - Read commands                │
│  - Execute commands             │
│  - Write responses              │
└──────────┬──────────────────────┘
           ↓
┌─────────────────────────────────┐
│      In-Memory Data             │
│   (Hash Tables, Skip Lists)     │
└─────────────────────────────────┘
```

### Why Single-Threaded?
- No lock contention
- Simple implementation
- CPU is rarely the bottleneck (memory/network bound)
- Can handle 100K+ ops/sec on single thread

### Multi-Threading (Redis 6.0+)
```
┌──────────────────────────────────┐
│   I/O Threads (Read/Write)       │
│   Thread 1, 2, 3, 4              │
└──────────┬───────────────────────┘
           ↓
┌──────────────────────────────────┐
│   Main Thread (Command Execution)│
│   Single-threaded                │
└──────────────────────────────────┘
```

## Data Structures

### 1. Simple Dynamic String (SDS)
```c
// Redis custom string implementation
struct sdshdr {
    int len;        // Current length
    int free;       // Available space
    char buf[];     // Actual data
};

// Benefits:
// - O(1) length lookup
// - Binary safe (can store any data)
// - Prevents buffer overflow
```

**Example:**
```python
import redis
r = redis.Redis()

# Internally stored as SDS
r.set('name', 'John')
r.append('name', ' Doe')  # O(1) append
print(r.strlen('name'))    # O(1) length
```

### 2. Hash Table (Dict)
```c
// Two hash tables for rehashing
typedef struct dict {
    dictht ht[2];           // Two hash tables
    long rehashidx;         // Rehashing progress (-1 if not rehashing)
    int iterators;          // Number of active iterators
} dict;

typedef struct dictht {
    dictEntry **table;      // Hash table array
    unsigned long size;     // Size of table
    unsigned long used;     // Number of entries
} dictht;
```

**Progressive Rehashing:**
```
When hash table grows:
1. Allocate new larger table (ht[1])
2. Gradually move entries from ht[0] to ht[1]
3. Each operation moves a few entries
4. No blocking - incremental rehashing
```

**Example:**
```python
# Hash operations
r.hset('user:1000', mapping={
    'name': 'John',
    'age': 30,
    'email': 'john@example.com'
})

r.hget('user:1000', 'name')      # O(1)
r.hgetall('user:1000')           # O(N)
r.hincrby('user:1000', 'age', 1) # Atomic increment
```

### 3. Skip List (Sorted Set)
```
Level 3:  1 ─────────────────────→ 50
Level 2:  1 ──────→ 20 ──────────→ 50
Level 1:  1 ──→ 10 → 20 ──→ 30 ──→ 50
Level 0:  1 → 5 → 10 → 20 → 25 → 30 → 40 → 50

Search complexity: O(log N)
Insert complexity: O(log N)
Range queries: O(log N + M)
```

**Why Skip List over Balanced Tree?**
- Simpler implementation
- Better cache locality
- Easier to implement range queries
- Lock-free in concurrent scenarios

**Example:**
```python
# Sorted set (leaderboard)
r.zadd('leaderboard', {
    'player1': 1000,
    'player2': 1500,
    'player3': 2000
})

r.zrange('leaderboard', 0, -1, withscores=True)  # Get all
r.zrevrange('leaderboard', 0, 9)                 # Top 10
r.zrangebyscore('leaderboard', 1000, 2000)       # Range query
```

### 4. Linked List
```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct list {
    listNode *head;
    listNode *tail;
    unsigned long len;
} list;
```

**Example:**
```python
# List operations
r.lpush('queue', 'task1', 'task2', 'task3')  # Push left
r.rpush('queue', 'task4')                     # Push right
r.lpop('queue')                               # Pop left
r.lrange('queue', 0, -1)                      # Get all
```

### 5. Integer Set (IntSet)
```c
// Space-efficient for small integer sets
typedef struct intset {
    uint32_t encoding;  // int16, int32, or int64
    uint32_t length;
    int8_t contents[];  // Sorted array
} intset;

// Automatically upgrades encoding when needed
```

**Example:**
```python
# Small integer set uses intset
r.sadd('numbers', 1, 2, 3, 4, 5)
# Internally: intset (memory efficient)

# Large or non-integer set uses hash table
r.sadd('users', 'user1', 'user2', 'user3')
# Internally: hash table
```

### 6. Zip List (Compressed List)
```
Compact representation for small lists/hashes
┌──────┬──────┬──────┬──────┬──────┐
│ len1 │ val1 │ len2 │ val2 │ end  │
└──────┴──────┴──────┴──────┴──────┘

Benefits:
- Contiguous memory (cache-friendly)
- No pointers overhead
- Used for small hashes, lists, sorted sets
```

**Example:**
```python
# Small hash uses ziplist
r.hset('small:hash', mapping={'a': '1', 'b': '2'})
# Internally: ziplist (memory efficient)

# Large hash uses hash table
for i in range(1000):
    r.hset('large:hash', f'key{i}', f'value{i}')
# Internally: hash table
```

## Memory Management

### Object Encoding
```c
typedef struct redisObject {
    unsigned type:4;        // String, List, Set, Hash, Zset
    unsigned encoding:4;    // How data is stored
    unsigned lru:24;        // LRU time or LFU data
    int refcount;           // Reference counting
    void *ptr;              // Pointer to actual data
} robj;
```

### Encoding Types
```python
# String encodings
r.set('int', '123')           # int encoding
r.set('embstr', 'short')      # embstr (≤44 bytes)
r.set('raw', 'x' * 100)       # raw (>44 bytes)

# Check encoding
r.object('encoding', 'int')   # "int"

# List encodings
r.lpush('small', 1, 2, 3)     # ziplist
r.lpush('large', *range(1000)) # linkedlist

# Hash encodings
r.hset('small', 'a', '1')     # ziplist
r.hset('large', mapping={f'k{i}': f'v{i}' for i in range(1000)})  # hashtable
```

### Memory Optimization
```python
# Check memory usage
r.memory_usage('mykey')

# Memory stats
info = r.info('memory')
print(f"Used memory: {info['used_memory_human']}")
print(f"Peak memory: {info['used_memory_peak_human']}")

# Set max memory
r.config_set('maxmemory', '2gb')

# Eviction policies
r.config_set('maxmemory-policy', 'allkeys-lru')
```

### Eviction Policies
```
noeviction      - Return error when memory full
allkeys-lru     - Evict least recently used keys
allkeys-lfu     - Evict least frequently used keys
allkeys-random  - Evict random keys
volatile-lru    - Evict LRU keys with TTL
volatile-lfu    - Evict LFU keys with TTL
volatile-random - Evict random keys with TTL
volatile-ttl    - Evict keys with shortest TTL
```

## Persistence

### 1. RDB (Redis Database Backup)
```
Point-in-time snapshots
┌─────────────────────────────────┐
│  Memory State at Time T         │
│  ↓                               │
│  Binary Snapshot File (dump.rdb)│
└─────────────────────────────────┘

Triggers:
- SAVE (blocking)
- BGSAVE (background fork)
- Automatic (save 900 1, save 300 10)
```

**Configuration:**
```python
# Automatic snapshots
r.config_set('save', '900 1 300 10 60 10000')
# Save if: 1 change in 900s, 10 in 300s, 10000 in 60s

# Manual snapshot
r.save()      # Blocking
r.bgsave()    # Background (fork)

# Check last save
r.lastsave()
```

**RDB Process:**
```
1. Redis forks child process
2. Child writes snapshot to temp file
3. Child renames temp file to dump.rdb
4. Parent continues serving requests

Pros:
- Compact single file
- Fast restart
- Good for backups

Cons:
- Data loss between snapshots
- Fork can be slow with large dataset
```

### 2. AOF (Append Only File)
```
Log of all write operations
┌─────────────────────────────────┐
│ SET key1 value1                 │
│ LPUSH list1 item1               │
│ HSET hash1 field1 value1        │
│ DEL key2                        │
└─────────────────────────────────┘

Replay on restart to rebuild state
```

**Configuration:**
```python
# Enable AOF
r.config_set('appendonly', 'yes')

# Sync policy
r.config_set('appendfsync', 'everysec')
# Options: always, everysec, no

# AOF rewrite (compact log)
r.bgrewriteaof()
```

**AOF Sync Policies:**
```
always   - Fsync after every write (slow, safest)
everysec - Fsync every second (good balance)
no       - Let OS decide (fast, least safe)
```

**AOF Rewrite:**
```
Original AOF:
SET counter 1
INCR counter
INCR counter
INCR counter

After rewrite:
SET counter 4

Triggered automatically or manually
```

### 3. Hybrid Persistence (RDB + AOF)
```python
# Redis 4.0+ combines both
r.config_set('aof-use-rdb-preamble', 'yes')

# AOF file contains:
# 1. RDB snapshot (fast load)
# 2. AOF commands since snapshot (no data loss)
```

## Replication

### Master-Slave Architecture
```
┌──────────────┐
│    Master    │ (Read/Write)
└──────┬───────┘
       │
   ┌───┴───┬───────┐
   ↓       ↓       ↓
┌──────┐ ┌──────┐ ┌──────┐
│Slave1│ │Slave2│ │Slave3│ (Read-only)
└──────┘ └──────┘ └──────┘
```

### Replication Process
```
1. Slave connects to master
2. Slave sends SYNC command
3. Master starts BGSAVE (RDB snapshot)
4. Master buffers new commands
5. Master sends RDB to slave
6. Slave loads RDB
7. Master sends buffered commands
8. Continuous replication (streaming)
```

**Configuration:**
```python
# On slave
r.slaveof('master_host', 6379)

# Check replication status
info = r.info('replication')
print(info['role'])  # master or slave
print(info['connected_slaves'])

# Promote slave to master
r.slaveof('NO', 'ONE')
```

### Partial Resynchronization
```
Master maintains replication backlog
┌─────────────────────────────────┐
│  Circular buffer of commands    │
│  (default 1MB)                  │
└─────────────────────────────────┘

If slave disconnects briefly:
- Slave sends offset
- Master sends missing commands
- No full resync needed
```

## Transactions

### MULTI/EXEC
```python
# Atomic transaction
pipe = r.pipeline()
pipe.multi()
pipe.set('key1', 'value1')
pipe.incr('counter')
pipe.lpush('list', 'item')
pipe.execute()  # All or nothing

# Commands queued, executed atomically
```

### WATCH (Optimistic Locking)
```python
# Watch key for changes
r.watch('balance')

balance = int(r.get('balance'))
if balance >= 100:
    pipe = r.pipeline()
    pipe.multi()
    pipe.decrby('balance', 100)
    pipe.incrby('spent', 100)
    pipe.execute()
else:
    r.unwatch()

# If 'balance' changed between WATCH and EXEC, transaction aborted
```

## Pub/Sub

### Architecture
```
┌──────────┐     ┌──────────┐     ┌──────────┐
│Publisher1│────→│  Redis   │────→│Subscriber│
└──────────┘     │ Channels │     └──────────┘
┌──────────┐     │          │     ┌──────────┐
│Publisher2│────→│          │────→│Subscriber│
└──────────┘     └──────────┘     └──────────┘
```

**Example:**
```python
# Subscriber
import redis

r = redis.Redis()
pubsub = r.pubsub()
pubsub.subscribe('news', 'updates')

for message in pubsub.listen():
    if message['type'] == 'message':
        print(f"Channel: {message['channel']}")
        print(f"Data: {message['data']}")

# Publisher
r.publish('news', 'Breaking news!')
r.publish('updates', 'New version released')
```

### Pattern Matching
```python
# Subscribe to pattern
pubsub.psubscribe('news.*', 'user.*')

# Matches: news.sports, news.tech, user.login, user.logout
```

## Lua Scripting

### Atomic Script Execution
```python
# Lua script runs atomically
script = """
local current = redis.call('GET', KEYS[1])
if not current then
    return redis.call('SET', KEYS[1], ARGV[1])
elseif tonumber(current) < tonumber(ARGV[1]) then
    return redis.call('SET', KEYS[1], ARGV[1])
else
    return 0
end
"""

# Execute script
r.eval(script, 1, 'max_value', 100)

# Register script (returns SHA)
sha = r.script_load(script)
r.evalsha(sha, 1, 'max_value', 150)
```

### Benefits
- Atomic execution
- Reduced network round trips
- Complex logic on server side

## Clustering

### Redis Cluster Architecture
```
┌────────────────────────────────────┐
│   16384 Hash Slots                 │
│   Distributed across nodes         │
└────────────────────────────────────┘
         ↓           ↓           ↓
    ┌────────┐  ┌────────┐  ┌────────┐
    │ Node 1 │  │ Node 2 │  │ Node 3 │
    │ 0-5460 │  │5461-   │  │10923-  │
    │        │  │10922   │  │16383   │
    └────────┘  └────────┘  └────────┘
         ↓           ↓           ↓
    ┌────────┐  ┌────────┐  ┌────────┐
    │Replica1│  │Replica2│  │Replica3│
    └────────┘  └────────┘  └────────┘
```

### Hash Slot Calculation
```python
# CRC16(key) % 16384
slot = crc16('mykey') % 16384

# Hash tags for multi-key operations
r.set('{user:1000}:name', 'John')
r.set('{user:1000}:age', 30)
# Both keys go to same slot (hash on 'user:1000')
```

### Cluster Operations
```python
from rediscluster import RedisCluster

# Connect to cluster
rc = RedisCluster(startup_nodes=[
    {'host': '127.0.0.1', 'port': 7000},
    {'host': '127.0.0.1', 'port': 7001},
    {'host': '127.0.0.1', 'port': 7002}
])

# Operations automatically routed to correct node
rc.set('key1', 'value1')
rc.get('key1')

# Multi-key operations require hash tags
rc.mget('{user:1}:name', '{user:1}:age')
```

## Performance Optimization

### Pipeline
```python
# Without pipeline (4 round trips)
r.set('key1', 'value1')
r.set('key2', 'value2')
r.set('key3', 'value3')
r.set('key4', 'value4')

# With pipeline (1 round trip)
pipe = r.pipeline()
pipe.set('key1', 'value1')
pipe.set('key2', 'value2')
pipe.set('key3', 'value3')
pipe.set('key4', 'value4')
pipe.execute()

# 4x faster for network-bound operations
```

### Connection Pooling
```python
# Reuse connections
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=50
)

r = redis.Redis(connection_pool=pool)
```

### Lazy Free (Async Delete)
```python
# Blocking delete (can freeze Redis)
r.delete('huge_key')  # Blocks until deleted

# Non-blocking delete (Redis 4.0+)
r.unlink('huge_key')  # Returns immediately, deletes in background
```

## Monitoring

### INFO Command
```python
# Server info
info = r.info()
print(f"Version: {info['redis_version']}")
print(f"Uptime: {info['uptime_in_seconds']}")
print(f"Connected clients: {info['connected_clients']}")
print(f"Used memory: {info['used_memory_human']}")
print(f"Total commands: {info['total_commands_processed']}")
print(f"Ops per sec: {info['instantaneous_ops_per_sec']}")
```

### SLOWLOG
```python
# Get slow queries
slowlog = r.slowlog_get(10)
for entry in slowlog:
    print(f"Command: {entry['command']}")
    print(f"Duration: {entry['duration']} microseconds")
```

### MONITOR (Debug)
```python
# Real-time command stream (expensive!)
# Use only for debugging
for command in r.monitor():
    print(command)
```

## Best Practices

1. **Use appropriate data structures**
2. **Enable persistence (AOF + RDB)**
3. **Set maxmemory and eviction policy**
4. **Use pipelining for bulk operations**
5. **Monitor memory usage and slow queries**
6. **Use connection pooling**
7. **Avoid KEYS command in production (use SCAN)**
8. **Use Lua scripts for complex atomic operations**
9. **Set appropriate TTLs**
10. **Use Redis Cluster for horizontal scaling**

## Conclusion

Redis internals reveal a carefully designed system optimizing for speed through single-threaded execution, efficient data structures (SDS, skip lists, zip lists), smart memory management, and flexible persistence options. Understanding these internals helps make better architectural decisions and optimize Redis usage.
