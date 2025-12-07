# Database Deadlocks with Docker PostgreSQL

## What is a Deadlock?

A deadlock occurs when two or more transactions are waiting for each other to release locks, creating a circular dependency. Neither transaction can proceed, resulting in a standstill.

**Example Scenario:**
- Transaction 1 locks Row A, wants Row B
- Transaction 2 locks Row B, wants Row A
- Both wait forever → Deadlock!

## Setup Docker PostgreSQL

```bash
# Start PostgreSQL container
docker run --name postgres-deadlock -e POSTGRES_PASSWORD=postgres -p 5432:5432 -d postgres

# Access PostgreSQL terminal
docker exec -it postgres-deadlock psql -U postgres postgres
```

## Prepare Test Database

```sql
-- Create test table
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    balance DECIMAL(10,2)
);

-- Insert sample data
INSERT INTO accounts (name, balance) VALUES 
    ('Alice', 1000.00),
    ('Bob', 1500.00);

-- Verify data
SELECT * FROM accounts;
```

## Deadlock Demonstration

### Method 1: Classic Two-Transaction Deadlock

**Terminal 1 (Transaction 1):**
```sql
-- Start transaction
BEGIN;

-- Lock account 1
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Transaction 1 now holds lock on row id=1

-- Wait 10 seconds before next step
SELECT pg_sleep(10);

-- Try to lock account 2 (will wait if Transaction 2 locked it)
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
```

**Terminal 2 (Transaction 2) - Execute within 10 seconds:**
```sql
-- Start transaction
BEGIN;

-- Lock account 2
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
-- Transaction 2 now holds lock on row id=2

-- Try to lock account 1 (will cause DEADLOCK)
UPDATE accounts SET balance = balance + 50 WHERE id = 1;

COMMIT;
```

**Result:** PostgreSQL detects the deadlock and aborts one transaction with:
```
ERROR:  deadlock detected
DETAIL:  Process X waits for ShareLock on transaction Y; blocked by process Z.
```

### Method 2: Using Docker Exec (Single Command)

**Terminal 1:**
```bash
docker exec -it postgres-deadlock psql -U postgres postgres -c "
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SELECT pg_sleep(15);
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
"
```

**Terminal 2 (run immediately):**
```bash
docker exec -it postgres-deadlock psql -U postgres postgres -c "
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
SELECT pg_sleep(2);
UPDATE accounts SET balance = balance + 50 WHERE id = 1;
COMMIT;
"
```

## Deadlock Detection Query

```sql
-- Check for locks
SELECT 
    pid,
    usename,
    pg_blocking_pids(pid) as blocked_by,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

## How PostgreSQL Handles Deadlocks

1. **Detection:** PostgreSQL checks for deadlocks periodically (default: 1 second)
2. **Resolution:** Aborts one transaction (victim selection based on cost)
3. **Error:** Returns `ERROR: deadlock detected`
4. **Retry:** Application should retry the aborted transaction

## Prevention Strategies

1. **Lock Order:** Always acquire locks in the same order
2. **Keep Transactions Short:** Minimize lock hold time
3. **Use Lower Isolation Levels:** When appropriate (READ COMMITTED)
4. **Timeout Settings:** Set `lock_timeout` and `deadlock_timeout`
5. **Explicit Locking:** Use `SELECT ... FOR UPDATE` to control lock order

```sql
-- Set lock timeout
SET lock_timeout = '5s';

-- Explicit lock ordering
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Now safe to update in any order
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

## Cleanup

```bash
# Stop and remove container
docker stop postgres-deadlock
docker rm postgres-deadlock
```
