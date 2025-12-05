# How UUIDs in B+Tree Indexes Affect Performance

## Quick Recap: What is a B+Tree?

A B+Tree is the **most common index structure** used by databases (MySQL, PostgreSQL, etc.).

Key properties:
- Data is stored in **sorted order**
- Leaf nodes are linked together (fast range scans)
- New data is inserted in its **sorted position**

---

## The Two Types of IDs

### Auto-Increment IDs (Sequential)
```
1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12...
```

Always increasing, predictable, sorted by creation time.

### UUIDs (Random)
```
550e8400-e29b-41d4-a716-446655440000
6ba7b810-9dad-11d1-80b4-00c04fd430c8
f47ac10b-58cc-4372-a567-0e02b2c3d479
```

Random, no order, scattered across the entire ID space.

---

## How Sequential IDs Work with B+Tree
```
INSERT id=1  → Goes to the end
INSERT id=2  → Goes to the end
INSERT id=3  → Goes to the end
INSERT id=4  → Goes to the end
...

B+Tree Leaf Nodes:

[1, 2, 3, 4] → [5, 6, 7, 8] → [9, 10, 11, 12] → [NEW INSERTS HERE]
                                                      ↑
                                            Always inserting here
```

**Result**: 
- Inserts always happen at the **rightmost leaf**
- That page stays in memory (hot page)
- Minimal page splits
- Sequential disk writes
- **FAST!**

---

## How Random UUIDs Work with B+Tree
```
INSERT uuid=f47a...  → Goes somewhere in the middle
INSERT uuid=123b...  → Goes near the beginning  
INSERT uuid=99cd...  → Goes somewhere else
INSERT uuid=550e...  → Goes to another random spot

B+Tree Leaf Nodes:

[123b, 1a4f, 2c3d] → [550e, 5f2a, 6ba7] → [99cd, a1b2, f47a]
       ↑                    ↑                     ↑
   Insert here          Insert here           Insert here
   
   RANDOM LOCATIONS EVERY TIME!
```

**Result**:
- Inserts happen at **random positions**
- Need to load random pages from disk
- Frequent page splits everywhere
- Random disk I/O (the slowest operation)
- **SLOW!**

---

## The Core Problems

### Problem 1: Random Disk I/O
```
Sequential ID:
  - Insert → Page already in memory → Fast write
  
Random UUID:
  - Insert → Which page? → Load from disk → Write → Repeat
  
Disk seek time: ~10ms
Memory access: ~0.0001ms

That's 100,000x slower!
```

### Problem 2: Page Splits

When a B+Tree page is full and you insert in the middle:
```
Before: [aaa, bbb, ccc, ddd] ← FULL!

Insert "bbc" (goes between bbb and ccc)

After:  [aaa, bbb] → [bbc, ccc, ddd]
              ↑
         PAGE SPLIT!
         
- Allocate new page
- Move half the data
- Update parent pointers
- Write multiple pages to disk
```

With UUIDs, page splits happen **constantly** because inserts are random.

With sequential IDs, page splits only happen at the **rightmost edge**.

### Problem 3: Poor Cache Utilization
```
Sequential IDs:
  - Recent data = Right side of tree
  - Queries for recent data = Cache hits
  - Buffer pool is used efficiently

Random UUIDs:
  - Recent data = Scattered everywhere
  - Queries touch random pages
  - Buffer pool thrashing
  - Cache misses everywhere
```

### Problem 4: Larger Index Size
```
Auto-increment (BIGINT): 8 bytes
UUID: 16 bytes (or 36 bytes as string!)

For 100 million rows:
  - BIGINT index: ~800 MB
  - UUID index: ~1.6 GB (or 3.6 GB as string)

Larger index = More disk reads = Slower queries
```

---

## Real Performance Impact

| Metric | Sequential ID | Random UUID |
|--------|---------------|-------------|
| Insert speed | Baseline | 2-10x slower |
| Index size | Baseline | 2-4x larger |
| Buffer pool efficiency | High | Low |
| Page splits | Rare | Constant |
| Disk I/O pattern | Sequential | Random |

At scale (millions of rows), UUIDs can make inserts **10x slower**.

---

## Solutions and Alternatives

### Option 1: Use Sequential IDs When Possible
```
Just use BIGINT AUTO_INCREMENT if you can.
Simple, fast, efficient.
```

### Option 2: UUID v7 (Time-Ordered)
```
UUID v7 structure:
[timestamp][random]

Example:
018f3b5c-1234-7abc-8def-0123456789ab
^^^^^^^^
 time-based prefix

Inserts become mostly sequential!
```

### Option 3: ULID (Universally Unique Lexicographically Sortable ID)
```
01ARZ3NDEKTSV4RRFFQ69G5FAV

- First 10 chars = timestamp (sorted by time)
- Last 16 chars = random
- Lexicographically sortable
```

### Option 4: Snowflake IDs
```
Structure: [timestamp][machine_id][sequence]

Used by: Twitter, Discord, Instagram

- Time-ordered
- Distributed generation
- 64-bit (same as BIGINT)
```

### Option 5: Composite Approach
```
- Primary Key: Auto-increment (for B+Tree efficiency)
- Public ID: UUID (for external APIs)

Table:
  id BIGINT PRIMARY KEY,        ← Internal, indexed
  public_id UUID UNIQUE,        ← External, rarely queried
  ...
```

---

## When UUIDs Make Sense

Despite the performance cost, use UUIDs when:

1. **Distributed systems** - Generate IDs without coordination
2. **Security** - IDs shouldn't be guessable
3. **Merging databases** - No ID conflicts
4. **Client-generated IDs** - Create ID before server round-trip

---

## Quick Decision Guide
```
Q: Do you need distributed ID generation?
   NO  → Use AUTO_INCREMENT
   YES ↓

Q: Do you need the ID to be unguessable?
   NO  → Use Snowflake ID
   YES ↓

Q: Can you use time-ordered UUIDs?
   YES → Use UUID v7 or ULID
   NO  → Use UUID v4 (accept the performance hit)
```

---

## Summary

| ID Type | B+Tree Friendly | Why |
|---------|-----------------|-----|
| Auto-increment | ✅ Excellent | Always appends to end |
| UUID v7 / ULID | ✅ Good | Mostly time-ordered |
| Snowflake | ✅ Good | Time-ordered, 64-bit |
| UUID v4 (random) | ❌ Poor | Random inserts everywhere |

**Bottom line**: Random UUIDs scatter inserts across the B+Tree, causing random disk I/O, page splits, and cache misses. Use time-ordered alternatives when possible.