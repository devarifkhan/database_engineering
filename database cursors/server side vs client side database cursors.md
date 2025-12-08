# Server-Side vs Client-Side Database Cursors

## Overview
Cursors can be executed either on the database server or on the client application, each with different performance and resource implications.

## Server-Side Cursors

### Definition
The cursor is created and maintained on the database server. The server holds the result set and sends rows to the client on demand.

### Characteristics
- Result set stays on the server
- Client fetches rows incrementally
- Server maintains cursor state and position
- Network traffic is minimized (only requested rows are sent)

### Advantages
- **Memory efficient on client**: Client doesn't need to load entire result set
- **Better for large result sets**: Ideal when dealing with millions of rows
- **Lower network traffic**: Only fetches what's needed
- **Supports scrolling**: Can move forward/backward through results

### Disadvantages
- **Server resource consumption**: Holds resources (memory, locks) on the server
- **Connection dependency**: Requires persistent connection
- **Scalability issues**: Multiple open cursors can strain server resources
- **Slower for small result sets**: Overhead of maintaining cursor state

### Use Cases
- Processing large datasets that don't fit in client memory
- Pagination through large result sets
- Long-running reports with incremental processing
- Streaming data to client applications

### Example (PostgreSQL)
```sql
BEGIN;
DECLARE my_cursor CURSOR FOR SELECT * FROM large_table;
FETCH 100 FROM my_cursor;  -- Fetch 100 rows at a time
CLOSE my_cursor;
COMMIT;
```

## Client-Side Cursors

### Definition
The entire result set is transferred to the client, and the client application manages the cursor operations in its own memory.

### Characteristics
- Full result set transferred to client immediately
- Client manages iteration and position
- Server releases resources after sending data
- All data resides in client memory

### Advantages
- **Server resources freed quickly**: No persistent state on server
- **Better scalability**: Server can handle more concurrent users
- **Faster for small result sets**: No round-trip overhead for each fetch
- **Offline processing**: Can work with data after disconnecting

### Disadvantages
- **High memory usage on client**: Must load entire result set
- **Network overhead**: All data transferred upfront
- **Not suitable for large datasets**: Can cause out-of-memory errors
- **Initial delay**: Must wait for complete result set transfer

### Use Cases
- Small to medium result sets
- Disconnected data processing
- When client needs random access to all rows
- Reporting with full dataset analysis

### Example (Python with psycopg2)
```python
import psycopg2

conn = psycopg2.connect("dbname=mydb")
cursor = conn.cursor()

# Client-side: fetches all rows immediately
cursor.execute("SELECT * FROM users")
all_rows = cursor.fetchall()  # All data in client memory

for row in all_rows:
    process(row)

cursor.close()
conn.close()
```

## Comparison Table

| Feature | Server-Side | Client-Side |
|---------|-------------|-------------|
| Memory usage (client) | Low | High |
| Memory usage (server) | High | Low |
| Network traffic | Minimal (on-demand) | High (all upfront) |
| Connection requirement | Persistent | Can disconnect |
| Large datasets | Excellent | Poor |
| Small datasets | Overhead | Efficient |
| Scalability | Limited | Better |
| Random access | Slower | Faster |

## Best Practices

### Use Server-Side When:
- Working with large result sets (millions of rows)
- Implementing pagination
- Memory is constrained on client
- Processing data incrementally

### Use Client-Side When:
- Result set is small (thousands of rows)
- Need random access to data
- Want to minimize server load
- Processing requires multiple passes over data

## Performance Considerations

**Server-Side:**
```sql
-- Good: Fetch in batches
FETCH 1000 FROM cursor;

-- Bad: Fetch one row at a time (too many round trips)
FETCH 1 FROM cursor;
```

**Client-Side:**
```python
# Good: Fetch all for small datasets
rows = cursor.fetchall()

# Bad: Using fetchall() on millions of rows
# Can cause memory exhaustion
```

## Inserting Million Rows with Python in Postgres using Client-Side Cursor

### Setup
```python
import psycopg2
import time

conn = psycopg2.connect(
    host="localhost",
    database="testdb",
    user="postgres",
    password="password"
)
cursor = conn.cursor()

# Create table
cursor.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100),
        email VARCHAR(100),
        age INTEGER
    )
""")
conn.commit()
```

### Method 1: Individual Inserts (Slow - Don't Use)
```python
start = time.time()

for i in range(1000000):
    cursor.execute(
        "INSERT INTO users (name, email, age) VALUES (%s, %s, %s)",
        (f"User{i}", f"user{i}@example.com", 25 + (i % 50))
    )
    conn.commit()  # Commit each insert

print(f"Time: {time.time() - start:.2f}s")  # ~hours
```

### Method 2: Batch Inserts (Better)
```python
start = time.time()
batch_size = 1000

for i in range(0, 1000000, batch_size):
    values = [
        (f"User{j}", f"user{j}@example.com", 25 + (j % 50))
        for j in range(i, min(i + batch_size, 1000000))
    ]
    
    cursor.executemany(
        "INSERT INTO users (name, email, age) VALUES (%s, %s, %s)",
        values
    )
    conn.commit()

print(f"Time: {time.time() - start:.2f}s")  # ~minutes
```

### Method 3: execute_batch (Faster)
```python
from psycopg2.extras import execute_batch

start = time.time()

data = [
    (f"User{i}", f"user{i}@example.com", 25 + (i % 50))
    for i in range(1000000)
]

execute_batch(
    cursor,
    "INSERT INTO users (name, email, age) VALUES (%s, %s, %s)",
    data,
    page_size=1000
)
conn.commit()

print(f"Time: {time.time() - start:.2f}s")  # ~30-60 seconds
```

### Method 4: COPY (Fastest)
```python
from io import StringIO

start = time.time()

# Create CSV data in memory
csv_data = StringIO()
for i in range(1000000):
    csv_data.write(f"User{i},user{i}@example.com,{25 + (i % 50)}\n")
csv_data.seek(0)

# Use COPY command
cursor.copy_from(
    csv_data,
    'users',
    columns=('name', 'email', 'age'),
    sep=','
)
conn.commit()

print(f"Time: {time.time() - start:.2f}s")  # ~5-10 seconds

cursor.close()
conn.close()
```

### Performance Comparison

| Method | Time (1M rows) | Notes |
|--------|----------------|-------|
| Individual inserts | Hours | Never use |
| executemany | Minutes | Acceptable |
| execute_batch | 30-60s | Good |
| COPY | 5-10s | Best |

### Best Practices for Bulk Inserts

1. **Use COPY for large datasets**: Fastest method, bypasses query parsing
2. **Disable indexes temporarily**: Drop indexes before insert, recreate after
3. **Use transactions**: Wrap inserts in single transaction
4. **Adjust batch size**: Balance memory usage vs performance (1000-10000)
5. **Disable autocommit**: Manual commit after batch completion

### Optimized Example with All Best Practices
```python
import psycopg2
from io import StringIO
import time

conn = psycopg2.connect(
    host="localhost",
    database="testdb",
    user="postgres",
    password="password"
)
conn.autocommit = False
cursor = conn.cursor()

# Drop indexes
cursor.execute("DROP INDEX IF EXISTS idx_users_email")

start = time.time()

# Generate and insert data using COPY
csv_data = StringIO()
for i in range(1000000):
    csv_data.write(f"User{i},user{i}@example.com,{25 + (i % 50)}\n")
csv_data.seek(0)

cursor.copy_from(csv_data, 'users', columns=('name', 'email', 'age'), sep=',')
conn.commit()

print(f"Insert time: {time.time() - start:.2f}s")

# Recreate indexes
start = time.time()
cursor.execute("CREATE INDEX idx_users_email ON users(email)")
conn.commit()
print(f"Index creation time: {time.time() - start:.2f}s")

cursor.close()
conn.close()
```

## Querying with Client-Side Cursor

### Basic Query
```python
import psycopg2

conn = psycopg2.connect(
    host="localhost",
    database="testdb",
    user="postgres",
    password="password"
)

# Client-side cursor (default)
cursor = conn.cursor()

# Execute query - all rows fetched to client immediately
cursor.execute("SELECT * FROM users WHERE age > 30")

# All data is already in client memory
rows = cursor.fetchall()
print(f"Total rows: {len(rows)}")

for row in rows:
    print(row)

cursor.close()
conn.close()
```

### Fetching Methods

#### fetchone() - One row at a time
```python
cursor.execute("SELECT * FROM users LIMIT 10")

while True:
    row = cursor.fetchone()
    if row is None:
        break
    print(row)
```

#### fetchmany() - Batch of rows
```python
cursor.execute("SELECT * FROM users")

while True:
    rows = cursor.fetchmany(100)  # Fetch 100 rows
    if not rows:
        break
    
    for row in rows:
        process(row)
```

#### fetchall() - All rows at once
```python
cursor.execute("SELECT * FROM users")
all_rows = cursor.fetchall()  # Entire result set in memory

for row in all_rows:
    print(row)
```

### Memory Considerations

```python
import psycopg2
import sys

conn = psycopg2.connect("dbname=testdb")
cursor = conn.cursor()

# Query 1 million rows
cursor.execute("SELECT * FROM users")  # Data transferred immediately

# Check memory usage
rows = cursor.fetchall()
print(f"Memory size: {sys.getsizeof(rows) / 1024 / 1024:.2f} MB")

# Problem: All 1M rows are in client memory
# Solution: Use server-side cursor for large datasets
```

### Iterating Over Results

```python
# Method 1: Using fetchall (loads all into memory)
cursor.execute("SELECT * FROM users")
for row in cursor.fetchall():
    print(row)

# Method 2: Cursor as iterator (still client-side, all data fetched)
cursor.execute("SELECT * FROM users")
for row in cursor:  # Iterates over already-fetched data
    print(row)
```

### Accessing Column Data

```python
# By index
cursor.execute("SELECT id, name, email FROM users LIMIT 5")
for row in cursor.fetchall():
    print(f"ID: {row[0]}, Name: {row[1]}, Email: {row[2]}")

# Using DictCursor for named access
from psycopg2.extras import DictCursor

cursor = conn.cursor(cursor_factory=DictCursor)
cursor.execute("SELECT id, name, email FROM users LIMIT 5")

for row in cursor.fetchall():
    print(f"ID: {row['id']}, Name: {row['name']}, Email: {row['email']}")

# Using RealDictCursor for dictionary access
from psycopg2.extras import RealDictCursor

cursor = conn.cursor(cursor_factory=RealDictCursor)
cursor.execute("SELECT id, name, email FROM users LIMIT 5")

for row in cursor.fetchall():
    print(row)  # {'id': 1, 'name': 'User1', 'email': 'user1@example.com'}
```

### Parameterized Queries

```python
# Safe from SQL injection
user_id = 100
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
row = cursor.fetchone()

# Multiple parameters
min_age = 25
max_age = 35
cursor.execute(
    "SELECT * FROM users WHERE age BETWEEN %s AND %s",
    (min_age, max_age)
)
rows = cursor.fetchall()
```

### Performance Tips for Client-Side Queries

```python
# 1. Limit result set size
cursor.execute("SELECT * FROM users LIMIT 1000")  # Don't fetch millions

# 2. Select only needed columns
cursor.execute("SELECT id, name FROM users")  # Not SELECT *

# 3. Use WHERE clauses to filter on server
cursor.execute("SELECT * FROM users WHERE age > 30")  # Filter server-side

# 4. Use indexes for faster queries
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))  # Indexed column

# 5. Close cursor when done
cursor.close()
```

### When Client-Side Cursor Fails

```python
# Problem: Query returns 10 million rows
try:
    cursor.execute("SELECT * FROM huge_table")  # Tries to fetch all
    rows = cursor.fetchall()  # MemoryError!
except MemoryError:
    print("Out of memory! Use server-side cursor instead")

# Solution: Use server-side cursor (next section)
```

### Complete Example

```python
import psycopg2
from psycopg2.extras import RealDictCursor

def query_users(min_age, max_age):
    conn = psycopg2.connect(
        host="localhost",
        database="testdb",
        user="postgres",
        password="password"
    )
    
    cursor = conn.cursor(cursor_factory=RealDictCursor)
    
    try:
        cursor.execute(
            "SELECT id, name, email, age FROM users WHERE age BETWEEN %s AND %s",
            (min_age, max_age)
        )
        
        # Process in batches to avoid memory issues
        while True:
            rows = cursor.fetchmany(1000)
            if not rows:
                break
            
            for row in rows:
                print(f"{row['name']}: {row['email']} (Age: {row['age']})")
        
    finally:
        cursor.close()
        conn.close()

query_users(25, 35)
```

## Conclusion

Choose based on your specific requirements:
- **Large datasets + limited client memory** → Server-side cursors
- **Small datasets + need for speed** → Client-side cursors
- **High concurrency** → Client-side cursors (less server load)
- **Streaming/pagination** → Server-side cursors
- **Bulk inserts** → Use COPY with client-side cursor
