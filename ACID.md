What are ACID properties in databases, and what is a database transaction? Explain both in detail.

**A transaction** is a group of database operations that must be treated as one complete unit—either everything succeeds (commit) or everything is undone (rollback). To ensure reliability, databases follow **ACID properties**: **Atomicity** means all steps execute fully or not at all; **Consistency** ensures the database remains valid before and after the transaction; **Isolation** makes sure multiple transactions don’t interfere with each other; and **Durability** guarantees that once a transaction is committed, the data is permanently saved even if the system crashes.


Here’s the same explanation rewritten in a **simple, natural, human tone**—no AI vibe, no robotic language.

---

# **Atomicity**

**Scenario:**
You’re sending **500 TK** from **Account A** to **Account B**.

A transaction for this transfer has only two steps:

1. Take 500 TK from Account A
2. Add that 500 TK to Account B

These two steps must happen together. If one fails, the whole thing must be undone.

---

## **Step-by-Step Example (covering both normal failure and system crash)**

### **➤ Transaction Starts**

```sql
BEGIN TRANSACTION;
```

### **➤ Step 1: Deduct 500 from Account A**

```sql
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
-- success
```

At this point, 500 has been taken from A.
Now two different things could happen next.

---

# **Case 1: A normal failure happens**

When trying to add 500 to Account B:

```sql
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
-- fails (maybe the row is locked or some rule is violated)
```

Since the second step didn’t work, the entire transaction is undone:

```sql
ROLLBACK;
```

### **Result**

* 500 goes back to Account A
* B doesn’t receive anything
* The database returns to the exact state before the transfer

This is **atomicity** — either all steps happen, or none of them happen.

---

# **Case 2: The database crashes right at that moment**

Imagine the system crashes **after the money is taken from A** but **before** it’s added to B.

When the database comes back up, it checks its transaction logs and sees that the transaction wasn’t completed.

So it automatically rolls everything back.

### **Result after restart**

* A gets its 500 back
* B’s balance stays the same
* No half-done changes are saved

This is atomicity again — even in a crash, the database won’t keep partial updates.



