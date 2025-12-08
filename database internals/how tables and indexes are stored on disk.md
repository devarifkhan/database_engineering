# How PostgreSQL Stores Tables and Indexes on Disk

A simple, high-level explanation of PostgreSQL's storage system.

---

## The Big Picture

Think of PostgreSQL storage like a **filing cabinet**:

```
Database = Filing Cabinet
Tables   = Folders inside the cabinet
Pages    = Sheets of paper in each folder
Rows     = Lines written on each sheet
```

---

## Tables: How Rows Are Stored

### Pages (The Basic Unit)

PostgreSQL doesn't save rows one by one. Instead, it groups rows into **pages** - fixed 8KB chunks.

```
┌─────────────────────────────────┐
│            Page                 │
│  ┌─────┐ ┌─────┐ ┌─────┐       │
│  │Row 1│ │Row 2│ │Row 3│  ...  │
│  └─────┘ └─────┘ └─────┘       │
│                                 │
│         (8KB total)             │
└─────────────────────────────────┘
```

**Why pages?**
- Reading from disk is slow
- Reading 8KB at once is almost as fast as reading 1 byte
- Caching works better with fixed-size chunks

### Heap Storage

PostgreSQL uses **heap storage** - rows are stored in no particular order.

```
INSERT new row → Find any page with free space → Put it there
```

It's like throwing papers into a box rather than organizing them alphabetically.

---

## Indexes: Finding Rows Quickly

Without an index, PostgreSQL must scan every row to find what you need (slow!).

### The Phone Book Analogy

```
Without Index (Heap Scan):
  "Find Ali's phone number"
  → Read every single entry until you find Ali
  → Could take forever in a big phone book

With Index (B-Tree):
  "Find Ali's phone number"
  → Go to 'A' section → Find 'Al' → Find 'Ali'
  → Just a few lookups
```

### How B-Tree Index Works

PostgreSQL's main index type is **B-Tree** - a balanced tree structure:

```
                    ┌─────────┐
                    │  M      │  ← Start here
                    └────┬────┘
                ┌────────┴────────┐
                ▼                 ▼
           ┌─────────┐       ┌─────────┐
           │  A-L    │       │  N-Z    │
           └────┬────┘       └────┬────┘
         ┌──────┴──────┐          │
         ▼             ▼          ▼
    ┌─────────┐   ┌─────────┐  ┌─────────┐
    │ Ali → 5 │   │ Kim → 8 │  │ Zara→12 │
    │ Bob → 2 │   │ Lee → 3 │  │         │
    └─────────┘   └─────────┘  └─────────┘
         │             │            │
         ▼             ▼            ▼
      Go to         Go to        Go to
      Page 5        Page 8       Page 12
```

**Key points:**
- Tree is always balanced (same depth everywhere)
- Leaf nodes point to actual table rows
- Finding any value takes the same number of steps

---

## How a Query Uses Storage

### Example: SELECT * FROM users WHERE email = 'ali@test.com'

**Without index:**
```
Page 1 → Scan all rows → No match
Page 2 → Scan all rows → No match
Page 3 → Scan all rows → Found it!
...
(Must check EVERY page)
```

**With index on email:**
```
Index: Jump to 'ali@test.com' → Points to Page 3, Row 5
Table: Go directly to Page 3, get Row 5
Done! (Just 2 lookups)
```

---

## MVCC: Multiple Versions of Rows

PostgreSQL keeps **old versions** of rows for concurrent access.

```
Transaction A reads row    Transaction B updates same row
        ↓                            ↓
   Sees version 1              Creates version 2
        ↓                            ↓
   Still sees version 1        Version 2 is committed
   (consistent snapshot)
```

**Trade-off:** Dead row versions pile up and need cleanup (VACUUM).

---

## Supporting Files

Each table has helper files:

| File | Purpose |
|------|---------|
| Main file | Actual row data |
| Free Space Map | Tracks which pages have room for new rows |
| Visibility Map | Tracks which pages are fully visible (helps VACUUM) |

---

## Large Values: TOAST

Values bigger than ~2KB are stored separately:

```
Main Table                    TOAST Storage
┌──────────────┐             ┌──────────────┐
│ id: 1        │             │              │
│ name: Ali    │             │  [Compressed │
│ bio: ────────┼────────────►│   large      │
│              │             │   text]      │
└──────────────┘             └──────────────┘
```

This keeps the main table compact and fast to scan.

---

## Summary

| Concept | Simple Explanation |
|---------|-------------------|
| **Pages** | Fixed 8KB containers holding multiple rows |
| **Heap** | Rows stored wherever there's space (unordered) |
| **B-Tree Index** | Organized tree for fast lookups |
| **MVCC** | Old row versions kept for concurrent readers |
| **VACUUM** | Cleanup process for dead rows |
| **TOAST** | Separate storage for large values |

---

## Key Takeaways

1. **Disk I/O is expensive** → PostgreSQL reads/writes in 8KB pages
2. **Indexes avoid scanning everything** → Tree structure for fast lookups
3. **Updates create new versions** → Old versions cleaned by VACUUM
4. **Large data stored separately** → Keeps main table fast

---

*The goal: Minimize disk reads while supporting concurrent access.*