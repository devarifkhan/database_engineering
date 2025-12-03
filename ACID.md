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



# **Isolation**

Isolation means:
**Even if many users run transactions at the same time, each transaction should behave as if it’s the only one running.**

Without isolation, people would see half-updated data, inconsistent values, or overwritten changes.

Databases handle this using **isolation levels**.

---

# **Read Phenomena (the problems isolation prevents)**

Below are the four common problems — all explained using **sales examples**.

---

## **1. Dirty Read — (Reading uncommitted data)**

**Meaning:**
One transaction reads data that another transaction has changed **but not committed yet**.

If that other transaction rolls back, the first transaction has already seen **wrong/temporary data**.

### **Sales Example**

* Transaction A updates a product price from **1000 TK → 800 TK**, but hasn’t committed yet.
* Transaction B reads the price and shows the customer **800 TK**.
* Transaction A fails and rolls back → the actual price stays **1000 TK**.

Transaction B relied on a value that was **never real**. That’s a dirty read.

---

## **2. Non-Repeatable Read — (Value changes between two reads)**

**Meaning:**
A transaction reads the same row twice, but the value changes in between because another transaction updated it.

### **Sales Example**

* Transaction A checks the price of a phone: **1000 TK**.
* Transaction B updates the price to **1100 TK** and commits.
* Transaction A checks the price again in the same session → now sees **1100 TK**.

Transaction A couldn’t “repeat” the same read.
The same row gave two different values.

---

## **3. Phantom Read — (New rows appear)**

**Meaning:**
A transaction reads a set of rows based on a condition, and on a later read, **new rows matching that condition appear** because another transaction inserted them.

### **Sales Example**

* Transaction A runs:
  “Show me all orders today above 500 TK.” → Gets **10 orders**.
* Transaction B inserts **2 new orders** above 500 TK and commits.
* Transaction A runs the same query again → now sees **12 orders**.

The extra rows are “phantoms” — they appear out of nowhere.

---

## **4. Lost Update — (One update overwrites another)**

**Meaning:**
Two transactions update the same value at the same time, and one update **overwrites** the other without knowing it.

### **Sales Example**

* Inventory for a product = **10 units**.
* Transaction A updates quantity to **8** after a sale.
* Transaction B updates quantity to **9** after a separate sale.
* Whichever commits last overwrites the other.

Final quantity may become **9**, even though **both sales happened**, and it should have been **7**.

This is a lost update.

---

# **Isolation Levels (from weakest to strongest)**

These control how much of the above problems are allowed or prevented.

| **Isolation Level**  | Dirty Read  | Non-Repeatable Read | Phantom Read | Lost Update |
| -------------------- | ----------- | ------------------- | ------------ | ----------- |
| **Read Uncommitted** | ❌ Allowed   | ❌ Allowed           | ❌ Allowed    | ❌ Allowed   |
| **Read Committed**   | ✔ Prevented | ❌ Allowed           | ❌ Allowed    | ❌ Allowed   |
| **Repeatable Read**  | ✔ Prevented | ✔ Prevented         | ❌ Allowed    | ❌ Allowed   |
| **Serializable**     | ✔ Prevented | ✔ Prevented         | ✔ Prevented  | ✔ Prevented |

### Quick summary:

* **Read Uncommitted** → “I don’t care, show me anything.”
* **Read Committed** → “Don’t show me uncommitted data.”
* **Repeatable Read** → “If I read a row, don’t let it change.”
* **Serializable** → “Treat everything like it’s single-threaded.”

---

# **Short, clean summary**

* **Isolation** protects transactions from interfering with each other.
* **Dirty reads**: reading uncommitted values.
* **Non-repeatable reads**: reading same row twice but value changes.
* **Phantom reads**: new rows appear between two queries.
* **Lost updates**: one update overwrites another.
* **Isolation levels** decide how much protection you get.
