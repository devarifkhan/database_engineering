
-- Database Cursors Explained:

-- A cursor is a database object that allows row-by-row processing of query results
-- Instead of processing the entire result set at once, cursors enable traversing results one row at a time

-- Example of declaring and using a cursor in SQL:

DECLARE employee_cursor CURSOR FOR 
SELECT employee_id, first_name, last_name 
FROM employees;

-- Variables to store the current row data
DECLARE @emp_id INT;
DECLARE @first_name VARCHAR(50);
DECLARE @last_name VARCHAR(50);

-- Open the cursor
OPEN employee_cursor;

-- Fetch the first row
FETCH NEXT FROM employee_cursor INTO 
    @emp_id,
    @first_name,
    @last_name;

-- Loop through the cursor
WHILE @@FETCH_STATUS = 0
BEGIN
    -- Process current row here
    PRINT 'Processing employee: ' + @first_name + ' ' + @last_name;
    
    -- Get next row
    FETCH NEXT FROM employee_cursor INTO 
        @emp_id,
        @first_name,
        @last_name;
END

-- Clean up
CLOSE employee_cursor;
DEALLOCATE employee_cursor;

-- Key Points About Cursors:
-- 1. DECLARE - Creates the cursor
-- 2. OPEN - Opens the cursor for use
-- 3. FETCH - Retrieves rows one at a time
-- 4. CLOSE - Closes the cursor
-- 5. DEALLOCATE - Removes cursor from memory

-- Note: Cursors can impact performance and should be used carefully
-- Consider set-based operations when possible instead of cursors