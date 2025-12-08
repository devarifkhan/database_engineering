

# 🔑 1. Primary Key (PK)

Most people think PK = unique ID.
But there’s a deeper truth:

### ✅ **Primary Key defines the *physical organization* of records in many databases.**

In traditional B-Tree row databases (MySQL InnoDB, PostgreSQL indexes, etc.):

* PK determines the **clustered index order**
* Data is **stored on disk sorted by the PK**
* All secondary indexes internally **point to the PK**, not to the raw row address

This has huge performance implications.

---

# 🔑 2. Secondary Key (SK)

Secondary keys = *additional* indexes to speed up lookups.

### But here’s the surprising part:

### ❗ Secondary keys **never** point directly to physical row locations.

Instead, they point to the **Primary Key value**, and then DB performs a second lookup.

This is called a **double lookup** or **bookmark lookup**.

---

# 🎯 Why this matters (the part most devs don’t know)

## 1️⃣ Storage difference

### Primary Key

* Stored only *once*
* Table is physically arranged by PK
* All secondary indexes reference PK

### Secondary Key

* Each entry stores:

  ```
  secondary_key_value + primary_key_value
  ```

So if your PK is large (UUID, long string):
👉 every secondary index becomes heavier
👉 inserts become slower
👉 IO increases

This is why performance experts prefer **auto-increment integer PKs**.

---

## 2️⃣ Query execution difference

### ✔ Query on Primary Key

Fastest possible lookup:

* One B-Tree traversal
* Immediately fetches row

### ✔ Query on Secondary Key

Needs two steps:

1. Find PK using secondary index
2. Use PK to fetch actual row

Imagine:

```sql
SELECT * FROM users WHERE email = 'arif@example.com';
```

If `email` is a secondary index → 2 lookups.

That’s why queries using secondary indexes are often slower.

---

## 3️⃣ Insert performance

### PK insert → Always maintains sorted order (cheap if auto-increment).

### SK insert → Updates all SK indexes + store PK reference

This creates extra cost.

So tables with 5–10 indexes?
👉 inserts/updates become slow even if CPU looks free.

---

## 4️⃣ Range Queries

* PK range = **very fast** (clustered, contiguous)
* SK range = slower (needs hopping between pages)

Example:

```sql
SELECT * FROM orders WHERE id BETWEEN 2000 AND 3000;
```

→ Super fast

```sql
SELECT * FROM orders WHERE amount BETWEEN 500 AND 1000;
```

→ Must jump around, not physically sequential

---

## 5️⃣ Cardinality matters

A PK must be:

* Unique
* Not NULL
* Stable (never changes)

A SK can:

* Have duplicates
* Be NULL
* Change frequently

---

# 🧠 Example that makes everything click

Suppose you have this table:

```
users
-----
id (PK)
email (SK)
phone (SK)
name
address
```

### What happens internally?

**Primary Key index (clustered):**

```
1 → Row data
2 → Row data
3 → Row data
```

**Secondary Index – email:**

```
email → PK
"arif@example.com" → 3
"sami@gmail.com"   → 1
"tuhin@yahoo.com"  → 2
```

**Query:**

```sql
SELECT * FROM users WHERE email = 'arif@example.com';
```

Steps:

1. Search email index → find PK = 3
2. Go to PK index → fetch row 3

This is the “double lookup” most people don't realize.

---

# 🧩 Summary Table

| Feature                      | Primary Key    | Secondary Key                    |
| ---------------------------- | -------------- | -------------------------------- |
| Uniqueness                   | Must be unique | Can be duplicate                 |
| Can be NULL?                 | No             | Yes                              |
| Defines physical sort order? | Yes            | No                               |
| Lookup steps                 | 1              | 2                                |
| Used by other indexes        | Yes            | Yes                              |
| Good for range queries       | Excellent      | Medium                           |
| Storage size impact          | Light          | Heavy (because stores PK inside) |

---

# 🎤 Final line:

A Primary Key isn’t just a unique column.
It **defines the physical structure** of your table.
Secondary Keys aren’t just “extra indexes”—they rely on the PK and add lookup + storage overhead.

That’s “what people usually don’t know.”

