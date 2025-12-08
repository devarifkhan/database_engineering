# **What is a Phantom Read?**

A **phantom read** happens when:

* Transaction A runs a query (e.g., count rows, list rows).
* Transaction B inserts or deletes rows that match the same query condition.
* Transaction A re-runs the same query **and sees new rows (“phantoms”)**.

It’s not “changing existing rows” — it’s **new rows appearing out of nowhere** during the same transaction.

---

# 🔥 **Practical PostgreSQL Example using Docker**

### **Step 1 — Start PostgreSQL with Docker**

```bash
docker run --name pg -e POSTGRES_PASSWORD=pass -p 5432:5432 -d postgres:16
```

---

### **Step 2 — Create a test table**

Connect:

```bash
docker exec -it pg psql -U postgres
```

Create the table:

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    price INT
);

INSERT INTO products (name, price) VALUES
('Chair', 100),
('Table', 200);
```

---

# 🧪 **Simulation of Phantom Read**

We will open **two sessions**:

* **Session A** → Long transaction
* **Session B** → Inserts a new product while A is still running

---

## **▶️ Session A (Terminal 1)**

Start a transaction with **READ COMMITTED** (default in PostgreSQL):

```sql
BEGIN;
```

Run a query:

```sql
SELECT * FROM products WHERE price > 50;
```

Output:

```
 id |  name  | price
----+--------+-------
 1  | Chair  | 100
 2  | Table  | 200
```

Keep this transaction open.

---

## **▶️ Session B (Terminal 2)**

While Session A is still running:

```sql
BEGIN;

INSERT INTO products (name, price)
VALUES ('Sofa', 300);

COMMIT;
```

Now a new row exists in the table.

---

## **▶️ Session A Again (Terminal 1)**

Run the same query:

```sql
SELECT * FROM products WHERE price > 50;
```

Output **now includes a new row**:

```
 id |  name  | price
----+--------+-------
 1  | Chair  | 100
 2  | Table  | 200
 3  | Sofa   | 300  <-- Phantom row
```

### 🔥 **This is a Phantom Read**

A new row appeared inside the same transaction of Session A.

---

# ⚡ How to Prevent Phantom Reads in PostgreSQL

Use **REPEATABLE READ** or **SERIALIZABLE** isolation.

### Example:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
```

Then Session A **would NOT see the new “Sofa” row**.

---

# Real-Life Example

Imagine an **e-commerce inventory dashboard**:

* Manager opens a transaction (Session A) to calculate total products priced over 50.
* While he is calculating, another employee (Session B) **adds a new product** priced 300.
* Manager recalculates and suddenly sees:

  * Product count increased
  * Total price increased

This extra product that appears is the **phantom**.

