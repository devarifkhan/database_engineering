# PostgreSQL ACID Properties

A hands-on guide to understanding ACID transactions using Docker and PostgreSQL.

## Setup

```bash
# Run PostgreSQL container
docker run --name postgres-acid -e POSTGRES_PASSWORD=secret -p 5432:5432 -d postgres

# Connect to the database
docker exec -it postgres-acid psql -U postgres
```

```sql
-- Create sample table
CREATE TABLE bank_account (
    id SERIAL PRIMARY KEY,
    account_name VARCHAR(50),
    balance NUMERIC
);

INSERT INTO bank_account (account_name, balance) 
VALUES ('Account A', 1000), ('Account B', 500);
```

---

## Atomicity

All or nothing. If any part of a transaction fails, the entire thing rolls back.

```sql
BEGIN;

UPDATE bank_account SET balance = balance - 200 WHERE account_name = 'Account A';
UPDATE bank_account SET balance = balance + 200 WHERE account_name = 'Account B';

-- Oops, let's cause an error
SELECT invalid_column FROM non_existent_table;
```

PostgreSQL now aborts the transaction. Any further commands return:

```
ERROR: current transaction is aborted, commands ignored until end of transaction block
```

```sql
ROLLBACK;

-- Verify nothing changed
SELECT * FROM bank_account;
```

Both accounts retain their original balances. The partial updates never happened.

### Successful Transaction

```sql
BEGIN;
UPDATE bank_account SET balance = balance - 200 WHERE account_name = 'Account A';
UPDATE bank_account SET balance = balance + 200 WHERE account_name = 'Account B';
COMMIT;

SELECT * FROM bank_account;
```

---

## Consistency

The database enforces rules. Transactions that violate constraints get rejected.

```sql
-- Add constraint: no negative balances
ALTER TABLE bank_account
ADD CONSTRAINT positive_balance CHECK (balance >= 0);
```

### Testing the Constraint

```sql
BEGIN;
-- Account A has $800, trying to withdraw $1000
UPDATE bank_account SET balance = balance - 1000 WHERE account_name = 'Account A';
```

```
ERROR: new row for relation "bank_account" violates check constraint "positive_balance"
DETAIL: Failing row contains (1, Account A, -200).
```

The database stays consistent—no overdrafts allowed.

---

## Isolation

Concurrent transactions don't interfere with each other.

Open **two terminals** and connect both to PostgreSQL:

```bash
docker exec -it postgres-acid psql -U postgres
```

**Terminal 1:**
```sql
BEGIN;
SELECT * FROM bank_account WHERE account_name = 'Account A';
UPDATE bank_account SET balance = balance - 100 WHERE account_name = 'Account A';
-- Don't commit yet, keep this open
```

**Terminal 2:**
```sql
BEGIN;
SELECT * FROM bank_account WHERE account_name = 'Account A';
UPDATE bank_account SET balance = balance - 200 WHERE account_name = 'Account A';
```

Terminal 2 **blocks**. It waits for Terminal 1 to finish. This prevents race conditions and dirty reads.

Commit in Terminal 1 to release the lock:
```sql
COMMIT;
```

---

## Durability

Committed transactions survive crashes.

```sql
BEGIN;
UPDATE bank_account SET balance = balance + 500 WHERE account_name = 'Account B';
COMMIT;
```

Kill the database:

```bash
docker stop postgres-acid
docker start postgres-acid
```

Check the data:

```bash
docker exec -it postgres-acid psql -U postgres -c "SELECT * FROM bank_account;"
```

The +500 is still there. Committed = permanent.

---

## Cleanup

```bash
docker stop postgres-acid
docker rm postgres-acid
```