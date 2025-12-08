
-- Exclusive Lock (X-Lock):
-- - Only one transaction can hold an exclusive lock on a resource at a time
-- - Prevents other transactions from reading or writing the locked resource
-- - Used for write operations (INSERT, UPDATE, DELETE)
-- Example:
SELECT * FROM employees WITH (XLOCK) 
WHERE employee_id = 100;

-- Shared Lock (S-Lock): 
-- - Multiple transactions can hold shared locks simultaneously
-- - Allows other transactions to read but not write to the locked resource
-- - Used for read operations (SELECT)
-- Example:
SELECT * FROM employees WITH (HOLDLOCK) 
WHERE department_id = 10;

-- Lock compatibility:
-- - Multiple shared locks can coexist
-- - Exclusive lock blocks both shared and exclusive locks
-- - Shared lock blocks exclusive locks but allows other shared locks

-- Transaction example with both lock types:
BEGIN TRANSACTION
    -- Get shared lock for reading
    SELECT * FROM orders WITH (HOLDLOCK)
    WHERE order_date = GETDATE()
    
    -- Get exclusive lock for updating
    UPDATE orders WITH (XLOCK)
    SET status = 'Processed'
    WHERE order_id = 500
COMMIT TRANSACTION