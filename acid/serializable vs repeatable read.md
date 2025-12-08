
# Scenario: Hospital — At least **1 doctor** must stay on duty

Table:

```
doctors_on_duty
----------------
doctor_id | on_duty (true/false)
```

Current state:

```
Dr A → true  
Dr B → true
```

Two doctors (Dr A and Dr B) decide to go off-duty at the same time.

We will see how the **same actions** behave differently in
**Repeatable Read** vs **Serializable**.

---

# Step-by-step Example

## **Transaction 1 (T1): Dr A**

## **Transaction 2 (T2): Dr B**

Both run the same logic:

```
BEGIN;  -- isolation level depends on example

SELECT COUNT(*) FROM doctors_on_duty WHERE on_duty = true;
-- result = 2 (Dr A and Dr B)

-- Since at least one doctor must stay,
-- they think it's safe to go off duty.

UPDATE doctors_on_duty
SET on_duty = false
WHERE doctor_id = 'A or B';

COMMIT;
```

---

# **Case 1: REPEATABLE READ**

### What happens?

* T1 and T2 start almost at the same time
* Both get a **snapshot showing 2 doctors on duty**
* Neither sees the other's UPDATE
* Both think: “Still 2 doctors → safe to go off duty”
* Both UPDATE and COMMIT successfully

### Final result (problem)

```
Dr A → false  
Dr B → false
```

**Zero doctors on duty → VALIDATION BROKEN.**
This is the **Write Skew anomaly**, which PostgreSQL *allows* under Repeatable Read.

---

# **Case 2: SERIALIZABLE**

### What happens?

* T1 and T2 start at the same time
* Both SELECT and see “2 doctors on duty”
* Both attempt to update

**PostgreSQL detects a dangerous conflict.**
If both succeed → rule is violated.

So PostgreSQL **aborts one transaction automatically**:

Example error:

```
ERROR: could not serialize access due to read/write dependencies among transactions
```

### Final state:

Only one transaction commits:

```
Dr A → false  
Dr B → true
```

**At least one doctor stays on duty → RULE PRESERVED.**

---

# Final Difference in One Line

| Isolation Level     | What Happens                                                |
| ------------------- | ----------------------------------------------------------- |
| **Repeatable Read** | Both transactions commit, rule is broken (write skew).      |
| **Serializable**    | One transaction is forced to abort → data stays consistent. |

---

# 10-second mental model

**Repeatable Read:**
“I see a frozen snapshot. I don’t care what others do, unless they modify exactly the same row.”

**Serializable:**
“I must ensure the result is as if we executed transactions one-by-one.
If not possible → abort someone.”
