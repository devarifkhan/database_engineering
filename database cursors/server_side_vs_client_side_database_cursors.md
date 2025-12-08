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

## Conclusion

Choose based on your specific requirements:
- **Large datasets + limited client memory** → Server-side cursors
- **Small datasets + need for speed** → Client-side cursors
- **High concurrency** → Client-side cursors (less server load)
- **Streaming/pagination** → Server-side cursors
- **Bulk inserts** → Use COPY with client-side cursor
