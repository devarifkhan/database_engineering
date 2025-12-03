What are ACID properties in databases, and what is a database transaction? Explain both in detail.

**A transaction** is a group of database operations that must be treated as one complete unit—either everything succeeds (commit) or everything is undone (rollback). To ensure reliability, databases follow **ACID properties**: **Atomicity** means all steps execute fully or not at all; **Consistency** ensures the database remains valid before and after the transaction; **Isolation** makes sure multiple transactions don’t interfere with each other; and **Durability** guarantees that once a transaction is committed, the data is permanently saved even if the system crashes.

