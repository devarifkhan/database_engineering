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

## Conclusion

Choose based on your specific requirements:
- **Large datasets + limited client memory** → Server-side cursors
- **Small datasets + need for speed** → Client-side cursors
- **High concurrency** → Client-side cursors (less server load)
- **Streaming/pagination** → Server-side cursors
