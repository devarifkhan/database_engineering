# Solving the Double Booking Problem

## The Problem

Multiple users trying to book the same resource (seat, room, ticket) simultaneously can result in double bookings due to race conditions.

**Scenario:**
```
User A: Check availability → Seat available → Book seat
User B: Check availability → Seat available → Book seat
Result: Same seat booked twice! ❌
```

## Setup

```sql
CREATE TABLE seats (
    seat_id INT PRIMARY KEY,
    seat_number VARCHAR(10),
    is_booked BOOLEAN DEFAULT FALSE
);

CREATE TABLE bookings (
    booking_id SERIAL PRIMARY KEY,
    seat_id INT REFERENCES seats(seat_id),
    user_name VARCHAR(50),
    booking_time TIMESTAMP DEFAULT NOW()
);

INSERT INTO seats (seat_id, seat_number, is_booked) VALUES
    (1, 'A1', FALSE),
    (2, 'A2', FALSE),
    (3, 'A3', FALSE);
```

## ❌ Wrong Approach (Race Condition)

```sql
-- User A
BEGIN;
SELECT * FROM seats WHERE seat_id = 1 AND is_booked = FALSE;  -- Available
-- User B checks here and also sees available
UPDATE seats SET is_booked = TRUE WHERE seat_id = 1;
INSERT INTO bookings (seat_id, user_name) VALUES (1, 'Alice');
COMMIT;

-- User B
BEGIN;
SELECT * FROM seats WHERE seat_id = 1 AND is_booked = FALSE;  -- Still sees available!
UPDATE seats SET is_booked = TRUE WHERE seat_id = 1;
INSERT INTO bookings (seat_id, user_name) VALUES (1, 'Bob');
COMMIT;
-- Double booking! ❌
```

## ✅ Solution 1: SELECT FOR UPDATE (Pessimistic Locking)

```sql
-- User A
BEGIN;
SELECT * FROM seats 
WHERE seat_id = 1 AND is_booked = FALSE 
FOR UPDATE;  -- Locks the row

-- User B waits here until User A commits
UPDATE seats SET is_booked = TRUE WHERE seat_id = 1;
INSERT INTO bookings (seat_id, user_name) VALUES (1, 'Alice');
COMMIT;

-- User B
BEGIN;
SELECT * FROM seats 
WHERE seat_id = 1 AND is_booked = FALSE 
FOR UPDATE;  -- Now gets the lock, but seat is booked
-- Returns empty (no available seat)
COMMIT;
```

**Advantages:**
- Prevents double booking
- Simple to implement
- Guaranteed consistency

**Disadvantages:**
- Reduced concurrency (locks held during transaction)
- Can cause deadlocks if not careful

## ✅ Solution 2: Atomic UPDATE with WHERE Condition

```sql
-- User A
BEGIN;
UPDATE seats 
SET is_booked = TRUE 
WHERE seat_id = 1 AND is_booked = FALSE
RETURNING *;  -- Returns row if updated, empty if already booked

-- Check if update succeeded
-- If RETURNING gives result, proceed
INSERT INTO bookings (seat_id, user_name) VALUES (1, 'Alice');
COMMIT;

-- User B (runs concurrently)
BEGIN;
UPDATE seats 
SET is_booked = TRUE 
WHERE seat_id = 1 AND is_booked = FALSE
RETURNING *;  -- Returns empty (already booked)
-- No rows updated, booking fails
ROLLBACK;
```

**Advantages:**
- No explicit locking
- Better concurrency
- Atomic operation

## ✅ Solution 3: SERIALIZABLE Isolation Level

```sql
-- User A
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT * FROM seats WHERE seat_id = 1 AND is_booked = FALSE;
UPDATE seats SET is_booked = TRUE WHERE seat_id = 1;
INSERT INTO bookings (seat_id, user_name) VALUES (1, 'Alice');
COMMIT;

-- User B
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT * FROM seats WHERE seat_id = 1 AND is_booked = FALSE;
UPDATE seats SET is_booked = TRUE WHERE seat_id = 1;
INSERT INTO bookings (seat_id, user_name) VALUES (1, 'Bob');
COMMIT;  -- ERROR: could not serialize access
```

**Advantages:**
- Strongest isolation guarantee
- Prevents all anomalies

**Disadvantages:**
- Performance overhead
- Requires retry logic for serialization failures

## ✅ Solution 4: Unique Constraint + Try-Catch

```sql
-- Add constraint
ALTER TABLE bookings ADD CONSTRAINT unique_seat_booking 
UNIQUE (seat_id);

-- User A
BEGIN;
INSERT INTO bookings (seat_id, user_name) VALUES (1, 'Alice');
UPDATE seats SET is_booked = TRUE WHERE seat_id = 1;
COMMIT;

-- User B
BEGIN;
INSERT INTO bookings (seat_id, user_name) VALUES (1, 'Bob');
-- ERROR: duplicate key value violates unique constraint
ROLLBACK;
```

**Advantages:**
- Database enforces uniqueness
- Simple and reliable

## Docker PostgreSQL Demo

**Terminal 1:**
```bash
docker exec -it postgres-deadlock psql -U postgres postgres

BEGIN;
SELECT * FROM seats WHERE seat_id = 1 AND is_booked = FALSE FOR UPDATE;
SELECT pg_sleep(10);  -- Simulate processing time
UPDATE seats SET is_booked = TRUE WHERE seat_id = 1;
INSERT INTO bookings (seat_id, user_name) VALUES (1, 'Alice');
COMMIT;
```

**Terminal 2 (run immediately):**
```bash
docker exec -it postgres-deadlock psql -U postgres postgres

BEGIN;
SELECT * FROM seats WHERE seat_id = 1 AND is_booked = FALSE FOR UPDATE;
-- Waits for Terminal 1 to commit
-- Then sees seat is already booked
COMMIT;
```

## Best Practice: Complete Booking Function

```sql
CREATE OR REPLACE FUNCTION book_seat(p_seat_id INT, p_user_name VARCHAR)
RETURNS TABLE(success BOOLEAN, message TEXT) AS $$
DECLARE
    v_updated INT;
BEGIN
    -- Atomic update with condition
    UPDATE seats 
    SET is_booked = TRUE 
    WHERE seat_id = p_seat_id AND is_booked = FALSE;
    
    GET DIAGNOSTICS v_updated = ROW_COUNT;
    
    IF v_updated = 0 THEN
        RETURN QUERY SELECT FALSE, 'Seat already booked or does not exist';
    ELSE
        INSERT INTO bookings (seat_id, user_name) VALUES (p_seat_id, p_user_name);
        RETURN QUERY SELECT TRUE, 'Booking successful';
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT * FROM book_seat(1, 'Alice');
SELECT * FROM book_seat(1, 'Bob');  -- Fails gracefully
```

## Comparison

| Solution | Concurrency | Complexity | Deadlock Risk |
|----------|-------------|------------|---------------|
| SELECT FOR UPDATE | Low | Low | Medium |
| Atomic UPDATE | High | Low | Low |
| SERIALIZABLE | Medium | Medium | Low |
| Unique Constraint | High | Low | Low |

## Recommended Approach

**For most cases:** Use **Atomic UPDATE with WHERE condition** (Solution 2)
- Best balance of simplicity and performance
- No explicit locking needed
- High concurrency

**For complex scenarios:** Combine **SELECT FOR UPDATE** with proper lock ordering to prevent deadlocks.

## Key Takeaway

Double booking is prevented by ensuring atomicity: check availability and book in a single atomic operation, or use explicit row-level locks to serialize access.
