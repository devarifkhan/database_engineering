# Database Connection Pooling

## What is Connection Pooling?

Connection pooling maintains a pool of reusable database connections instead of creating a new connection for each request.

**Without Pooling:**
```
Request → Open Connection → Query → Close Connection → Repeat
Cost: ~50-100ms per connection
```

**With Pooling:**
```
Request → Get Connection from Pool → Query → Return to Pool → Reuse
Cost: ~1ms to get connection
```

## Why Connection Pooling?

**Problems without pooling:**
- Opening connections is expensive (TCP handshake, authentication, SSL)
- Database has limited connections (PostgreSQL default: 100)
- High latency for each request
- Resource exhaustion under load

**Benefits with pooling:**
- ✓ Reuse existing connections (50-100x faster)
- ✓ Control max connections
- ✓ Better resource management
- ✓ Improved application performance

## Connection Lifecycle

```
[Idle] → [Borrowed] → [In Use] → [Returned] → [Idle]
                                      ↓
                              [Timeout/Error] → [Destroyed]
```

## PostgreSQL Connection Limits

```sql
-- Check max connections
SHOW max_connections;  -- Default: 100

-- Check current connections
SELECT count(*) FROM pg_stat_activity;

-- View active connections
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query
FROM pg_stat_activity;
```

## Docker PostgreSQL Setup

```bash
# Start PostgreSQL with custom max connections
docker run --name postgres-pool \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=testdb \
  -p 5432:5432 \
  -d postgres \
  -c max_connections=200

# Access PostgreSQL
docker exec -it postgres-pool psql -U postgres testdb
```

## Implementation Examples

### Python (psycopg2)

```python
from psycopg2 import pool

# Create connection pool
connection_pool = pool.SimpleConnectionPool(
    minconn=5,      # Minimum connections
    maxconn=20,     # Maximum connections
    host='localhost',
    database='testdb',
    user='postgres',
    password='postgres'
)

# Get connection from pool
def execute_query(query):
    conn = connection_pool.getconn()
    try:
        cursor = conn.cursor()
        cursor.execute(query)
        result = cursor.fetchall()
        return result
    finally:
        connection_pool.putconn(conn)  # Return to pool

# Usage
result = execute_query("SELECT * FROM users LIMIT 10")
```

### Python (SQLAlchemy)

```python
from sqlalchemy import create_engine

# Create engine with connection pool
engine = create_engine(
    'postgresql://postgres:postgres@localhost/testdb',
    pool_size=10,           # Number of connections to maintain
    max_overflow=20,        # Additional connections if pool exhausted
    pool_timeout=30,        # Wait time for available connection
    pool_recycle=3600,      # Recycle connections after 1 hour
    pool_pre_ping=True      # Verify connection before use
)

# Usage
with engine.connect() as conn:
    result = conn.execute("SELECT * FROM users")
    for row in result:
        print(row)
```

### Node.js (pg)

```javascript
const { Pool } = require('pg');

// Create connection pool
const pool = new Pool({
  host: 'localhost',
  database: 'testdb',
  user: 'postgres',
  password: 'postgres',
  port: 5432,
  max: 20,                // Maximum connections
  min: 5,                 // Minimum connections
  idleTimeoutMillis: 30000,  // Close idle connections after 30s
  connectionTimeoutMillis: 2000  // Wait 2s for connection
});

// Execute query
async function executeQuery(query) {
  const client = await pool.connect();
  try {
    const result = await client.query(query);
    return result.rows;
  } finally {
    client.release();  // Return to pool
  }
}

// Usage
executeQuery('SELECT * FROM users LIMIT 10')
  .then(rows => console.log(rows))
  .catch(err => console.error(err));
```

### Java (HikariCP)

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

// Configure connection pool
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/testdb");
config.setUsername("postgres");
config.setPassword("postgres");
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);
config.setConnectionTimeout(30000);
config.setIdleTimeout(600000);
config.setMaxLifetime(1800000);

HikariDataSource dataSource = new HikariDataSource(config);

// Execute query
try (Connection conn = dataSource.getConnection();
     Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {
    while (rs.next()) {
        System.out.println(rs.getString("name"));
    }
}
```

### Go (pgxpool)

```go
package main

import (
    "context"
    "github.com/jackc/pgx/v5/pgxpool"
)

func main() {
    // Create connection pool
    config, _ := pgxpool.ParseConfig(
        "postgres://postgres:postgres@localhost:5432/testdb?pool_max_conns=20&pool_min_conns=5",
    )
    
    pool, _ := pgxpool.NewWithConfig(context.Background(), config)
    defer pool.Close()
    
    // Execute query
    rows, _ := pool.Query(context.Background(), "SELECT * FROM users LIMIT 10")
    defer rows.Close()
    
    for rows.Next() {
        var id int
        var name string
        rows.Scan(&id, &name)
        println(id, name)
    }
}
```

## Key Configuration Parameters

| Parameter | Description | Typical Value |
|-----------|-------------|---------------|
| **pool_size** / **min** | Minimum connections | 5-10 |
| **max_connections** / **max** | Maximum connections | 10-50 |
| **pool_timeout** | Wait time for connection | 10-30s |
| **pool_recycle** | Recycle connection after | 1-2 hours |
| **idle_timeout** | Close idle connections | 10-30 minutes |
| **max_lifetime** | Max connection lifetime | 30-60 minutes |

## Sizing Your Pool

**Formula:**
```
pool_size = (core_count * 2) + effective_spindle_count
```

**Example:**
- 4 CPU cores
- 1 disk (effective_spindle_count = 1)
- Pool size = (4 * 2) + 1 = 9

**General Guidelines:**
- Start small (10-20 connections)
- Monitor and adjust based on load
- More connections ≠ better performance
- Too many connections cause contention

## Testing Connection Pool

```sql
-- Create test table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO users (name, email)
SELECT 
    'User ' || generate_series,
    'user' || generate_series || '@example.com'
FROM generate_series(1, 10000);
```

### Load Test Script (Python)

```python
import time
import threading
from psycopg2 import pool

# With pooling
connection_pool = pool.SimpleConnectionPool(5, 20,
    host='localhost', database='testdb',
    user='postgres', password='postgres'
)

def query_with_pool():
    conn = connection_pool.getconn()
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users LIMIT 100")
        cursor.fetchall()
    finally:
        connection_pool.putconn(conn)

# Run 100 concurrent queries
start = time.time()
threads = [threading.Thread(target=query_with_pool) for _ in range(100)]
for t in threads: t.start()
for t in threads: t.join()
print(f"With pooling: {time.time() - start:.2f}s")

# Without pooling (for comparison)
import psycopg2

def query_without_pool():
    conn = psycopg2.connect(
        host='localhost', database='testdb',
        user='postgres', password='postgres'
    )
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users LIMIT 100")
    cursor.fetchall()
    conn.close()

start = time.time()
threads = [threading.Thread(target=query_without_pool) for _ in range(100)]
for t in threads: t.start()
for t in threads: t.join()
print(f"Without pooling: {time.time() - start:.2f}s")
```

**Expected Results:**
```
With pooling: 0.5s
Without pooling: 5.0s
```

## Common Pitfalls

### 1. Pool Exhaustion
```python
# ❌ Wrong: Not returning connection
conn = pool.getconn()
cursor.execute(query)
# Connection never returned!

# ✅ Correct: Always return
conn = pool.getconn()
try:
    cursor.execute(query)
finally:
    pool.putconn(conn)
```

### 2. Too Many Connections
```python
# ❌ Wrong: Pool larger than DB limit
pool_size = 200  # PostgreSQL max_connections = 100

# ✅ Correct: Leave headroom
pool_size = 20   # Well below DB limit
```

### 3. Long-Running Transactions
```python
# ❌ Wrong: Holding connection too long
conn = pool.getconn()
process_for_10_minutes()  # Blocks other requests

# ✅ Correct: Release quickly
conn = pool.getconn()
data = fetch_data()
pool.putconn(conn)
process_for_10_minutes()  # Process after release
```

## Monitoring Pool Health

```sql
-- PostgreSQL: Check connection usage
SELECT 
    count(*) as total_connections,
    count(*) FILTER (WHERE state = 'active') as active,
    count(*) FILTER (WHERE state = 'idle') as idle
FROM pg_stat_activity;

-- Check connection age
SELECT 
    pid,
    usename,
    application_name,
    NOW() - backend_start as connection_age,
    state
FROM pg_stat_activity
ORDER BY backend_start;
```

## External Connection Poolers

### PgBouncer (Recommended)

```bash
# Install PgBouncer
docker run -d --name pgbouncer \
  -e DATABASES_HOST=postgres-pool \
  -e DATABASES_PORT=5432 \
  -e DATABASES_USER=postgres \
  -e DATABASES_PASSWORD=postgres \
  -e DATABASES_DBNAME=testdb \
  -e POOL_MODE=transaction \
  -e MAX_CLIENT_CONN=1000 \
  -e DEFAULT_POOL_SIZE=25 \
  -p 6432:6432 \
  edoburu/pgbouncer

# Connect through PgBouncer
psql -h localhost -p 6432 -U postgres testdb
```

**PgBouncer Modes:**
- **Session:** One connection per client session
- **Transaction:** Connection per transaction (most efficient)
- **Statement:** Connection per statement (rarely used)

## Key Takeaway

Connection pooling is essential for production applications. It reduces connection overhead by 50-100x, prevents resource exhaustion, and improves overall performance. Always use connection pooling with proper sizing (10-20 connections for most apps).
