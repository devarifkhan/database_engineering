# Two-Phase Locking (2PL)

## What is Two-Phase Locking?

Two-Phase Locking is a concurrency control protocol that ensures serializability by dividing transaction execution into two distinct phases:

1. **Growing Phase (Expanding):** Transaction can acquire locks but cannot release any
2. **Shrinking Phase (Contracting):** Transaction can release locks but cannot acquire new ones

**Lock Point:** The moment when the last lock is acquired (transition between phases)

## Why 2PL?

- Guarantees **conflict serializability**
- Prevents inconsistent reads and lost updates
- Standard protocol in most RDBMS (PostgreSQL, MySQL, Oracle)

## Basic 2PL Rules

```
✓ Acquire locks before accessing data
✓ Once a lock is released, no new locks can be acquired
✗ Cannot acquire lock after releasing any lock
```

## Example Timeline

```
Transaction T1:
Time    Action              Phase
----    ------              -----
t1      Lock(A)            Growing
t2      Read(A)            Growing
t3      Lock(B)            Growing  ← Lock Point
t4      Write(B)           Shrinking
t5      Unlock(A)          Shrinking
t6      Unlock(B)          Shrinking
t7      COMMIT             Shrinking
```

## Types of Two-Phase Locking

### 1. Basic 2PL (Conservative)
- Follows strict growing/shrinking phases
- **Problem:** Can still cause cascading aborts

### 2. Strict 2PL (S2PL)
- Holds all **exclusive locks** until COMMIT/ROLLBACK
- Prevents cascading aborts
- **Most commonly used** in databases

### 3. Rigorous 2PL
- Holds **all locks** (shared + exclusive) until COMMIT/ROLLBACK
- Simplest to implement
- Used in many commercial databases

## PostgreSQL Demonstration

```sql
-- Setup
CREATE TABLE inventory (
    product_id INT PRIMARY KEY,
    quantity INT
);

INSERT INTO inventory VALUES (1, 100), (2, 200);
```

### Strict 2PL in Action

**Transaction 1:**
```sql
BEGIN;

-- Growing Phase: Acquire locks
UPDATE inventory SET quantity = quantity - 10 WHERE product_id = 1;  -- Lock acquired
SELECT * FROM inventory WHERE product_id = 2 FOR UPDATE;             -- Lock acquired

-- Still in Growing Phase
UPDATE inventory SET quantity = quantity + 10 WHERE product_id = 2;

-- Shrinking Phase: All locks released at COMMIT
COMMIT;  -- Locks released here
```

**Transaction 2 (runs concurrently):**
```sql
BEGIN;

-- Will wait if T1 holds locks
UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 1;  -- Waits for T1

COMMIT;
```

## Visualizing Lock Acquisition

```
T1: ----[Lock A]----[Lock B]----[Work]----[COMMIT→Release All]----
T2: ----[Wait...]----[Wait...]----[Lock A]----[Lock B]----[COMMIT]----
```

## Docker PostgreSQL Example

```bash
# Terminal 1
docker exec -it postgres-deadlock psql -U postgres postgres

BEGIN;
UPDATE inventory SET quantity = quantity - 10 WHERE product_id = 1;
-- Lock held until COMMIT
SELECT pg_sleep(10);
COMMIT;  -- Lock released
```

```bash
# Terminal 2 (run immediately)
docker exec -it postgres-deadlock psql -U postgres postgres

BEGIN;
UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 1;
-- Waits for T1 to release lock
COMMIT;
```

## Check Active Locks

```sql
SELECT 
    locktype,
    relation::regclass,
    mode,
    transactionid,
    pid,
    granted
FROM pg_locks
WHERE pid = pg_backend_pid();
```

## 2PL vs Deadlocks

**Problem:** 2PL can cause deadlocks

```
T1: Lock(A) → Lock(B)
T2: Lock(B) → Lock(A)  ← Deadlock!
```

**Solution:** Deadlock detection and victim selection

## Advantages

✓ Guarantees serializability  
✓ Prevents dirty reads (with Strict 2PL)  
✓ Prevents lost updates  
✓ Industry standard  

## Disadvantages

✗ Can cause deadlocks  
✗ Reduced concurrency (locks held longer)  
✗ Potential for lock contention  

## Real-World Usage

- **PostgreSQL:** Uses Strict 2PL with MVCC
- **MySQL InnoDB:** Uses Strict 2PL
- **SQL Server:** Uses Strict 2PL
- **Oracle:** Uses MVCC with minimal locking

## Key Takeaway

Two-Phase Locking ensures database consistency by enforcing a simple rule: **acquire all locks before releasing any**. Most databases use Strict 2PL, holding locks until transaction completion to prevent cascading failures.
