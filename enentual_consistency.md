
# What is Eventual Consistency?

**Eventual consistency** means:

> Data is **not immediately the same** in all places,
> but if no new updates happen,
> **all systems will eventually reach the same value**.

It does **NOT** guarantee instant synchronization.
It only guarantees **eventual** synchronization.

---

# Real-Life Example (Your Phone Contacts)

You change your friend's name on your phone →
Google syncs →
Laptop syncs →
Tablet syncs.

They **don’t update at the same moment**.
But after some time, everything becomes consistent.

That is **eventual consistency**.

---

# System Example (Very Practical: Microservices + Kafka)

Imagine:

* **Order Service**
* **Inventory Service**
* **Notification Service**

When a user places an order:

1️⃣ Order Service writes to its database
2️⃣ Publishes an event: `order_created`
3️⃣ Inventory Service receives this event later
4️⃣ Reduces stock
5️⃣ Notification Service sends email later

If Inventory takes a few seconds to update, and Notification takes 5 seconds to send the email:

👉 **All services do NOT have the same state at the same time.**

But after the events propagate:

👉 **All systems eventually reflect the new order.**

This is **eventual consistency** in distributed systems.

---

# Banking Example (Highly Practical)

**You transfer money from bKash to bank:**

* bKash shows "Sent"
* Bank shows “pending”
* After a minute, bank updates the balance

The systems sync eventually, NOT instantly.
That’s eventual consistency.

---

# Why we use Eventual Consistency?

Because:

* Services are distributed
* Network has latency
* Real-time locking is expensive
* Systems need to scale horizontally
* Each service has its own DB (microservices)

Strong consistency = slow
Eventual consistency = fast + scalable

---

# 🎯 10-Second Summary

| Type                     | Meaning                                                                  |
| ------------------------ | ------------------------------------------------------------------------ |
| **Strong Consistency**   | Everyone sees the latest data instantly                                  |
| **Eventual Consistency** | Some see old data temporarily, but eventually all will see the same data |

