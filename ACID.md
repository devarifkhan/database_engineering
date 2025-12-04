# ACID Properties & Database Transactions

A **transaction** is a group of database operations treated as a single unit. Either everything succeeds (commit) or everything is undone (rollback).

ACID is what makes this reliable:

- **Atomicity** — all or nothing
- **Consistency** — rules are never broken
- **Isolation** — transactions don't interfere with each other
- **Durability** — committed data survives crashes

---

## Atomicity

You're transferring 500 TK from Account A to Account B.

Two steps:
1. Deduct 500 from A
2. Add 500 to B

Both must happen. If one fails, undo everything.

```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 500 WHERE id = 1;
-- success, 500 taken from A

UPDATE accounts SET balance = balance + 500 WHERE id = 2;
-- fails (row locked, constraint violated, whatever)

ROLLBACK;
```

Result: A gets its 500 back, B receives nothing. Database looks exactly like before.

### What if the system crashes mid-transaction?

Say the crash happens after deducting from A but before adding to B.

When the database restarts, it checks its transaction log, sees the transaction never completed, and rolls it back automatically.

Result: A gets its 500 back. No partial updates saved.

---

## Consistency

The database enforces rules. Transactions that would break those rules get rejected.

**Example rule:** Total sales must equal sum of all orders.

| Order ID | Amount |
|----------|--------|
| 1 | 100 TK |
| 2 | 200 TK |

Summary table shows: **300 TK** ✓

New order comes in: 150 TK

```sql
BEGIN TRANSACTION;
INSERT INTO orders (amount) VALUES (150);  -- orders now sum to 450
UPDATE summary SET total_sales = 450;      -- if this fails...
```

If step 2 fails, you'd have orders totaling 450 but summary showing 300. Inconsistent.

Consistency ensures: either both succeed, or neither. The database never ends up in a broken state.

---

## Isolation

Multiple transactions running simultaneously should behave as if they're running one at a time.

Without isolation, you get these problems:

### Dirty Read

Reading data that hasn't been committed yet.

- Transaction A changes product price: 1000 → 800 (not committed)
- Transaction B reads 800, shows it to customer
- Transaction A rolls back, price is still 1000

Transaction B showed a price that was never real.

### Non-Repeatable Read

Same row, different values on repeated reads.

- Transaction A reads phone price: 1000
- Transaction B updates price to 1100 and commits
- Transaction A reads again: 1100

Same query, two different answers within one transaction.

### Phantom Read

New rows appear between queries.

- Transaction A: "Show orders above 500 TK" → 10 results
- Transaction B inserts 2 new orders above 500 and commits
- Transaction A runs same query → 12 results

Rows appeared out of nowhere.

### Lost Update

Two updates, one gets overwritten.

- Inventory = 10
- Transaction A: sold 2 items, sets inventory to 8
- Transaction B: sold 1 item, sets inventory to 9
- Last commit wins

Final inventory: 9. But two sales happened, so it should be 7. One update is lost.

---

### Isolation Levels

From weakest to strongest:

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Lost Update |
|-------|------------|---------------------|--------------|-------------|
| Read Uncommitted | Allowed | Allowed | Allowed | Allowed |
| Read Committed | Prevented | Allowed | Allowed | Allowed |
| Repeatable Read | Prevented | Prevented | Allowed | Allowed |
| Serializable | Prevented | Prevented | Prevented | Prevented |

**Read Uncommitted** — no protection, see everything  
**Read Committed** — only see committed data  
**Repeatable Read** — rows you read won't change  
**Serializable** — full protection, behaves like single-threaded execution

---

## Durability

Once committed, data is permanent. Crashes don't erase it.

```sql
BEGIN TRANSACTION;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 42;
UPDATE sales SET total = total + 500;
COMMIT;
```

Power goes out immediately after commit.

When the database comes back up:
- Inventory still shows 1 less item
- Sales still includes the 500 TK

Committed means saved. Databases achieve this using write-ahead logs (WAL) that persist changes to disk before confirming the commit.

---

## Summary

| Property | What it guarantees |
|----------|-------------------|
| Atomicity | All steps complete or none do |
| Consistency | Database rules are never violated |
| Isolation | Concurrent transactions don't interfere |
| Durability | Committed data survives crashes |