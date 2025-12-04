Set Up PostgreSQL with Docker

docker run --name postgres-acid -e POSTGRES_PASSWORD=secret -p 5432:5432 -d postgres


Connect to PostgreSQL

docker exec -it postgres-acid psql -U postgres


Create a Sample Table

CREATE TABLE bank_account (
    id SERIAL PRIMARY KEY,
    account_name VARCHAR(50),
    balance NUMERIC
);

-- Insert initial data
INSERT INTO bank_account (account_name, balance) VALUES ('Account A', 1000), ('Account B', 500);


ACID


Start a transaction:
BEGIN;

Transfer funds from Account A to Account B:


UPDATE bank_account SET balance = balance - 200 WHERE account_name = 'Account A';

UPDATE bank_account SET balance = balance + 200 WHERE account_name = 'Account B';

Now, deliberately cause an error by attempting to select a non-existent table:

SELECT invalid_column FROM non_existent_table;

Since this operation failed, PostgreSQL automatically places the transaction in an aborted state. Any further commands issued in this transaction will trigger the following message:

current transaction is aborted, commands ignored until end of transaction block

To resolve this and roll back the transaction, simply run:
ROLLBACK;

This ensures that none of the changes (including the valid ones) are applied to the database, demonstrating atomicity—the “all or nothing” rule.

To verify that no changes were made, run the following:


SELECT * FROM bank_account;


You should see the original balances unchanged. Now, if you’d like to proceed with a successful transaction, you can begin a new transaction:

BEGIN;

Then, perform the transfer correctly:

UPDATE bank_account SET balance = balance - 200 WHERE account_name = 'Account A';
UPDATE bank_account SET balance = balance + 200 WHERE account_name = 'Account B';

And finally:

COMMIT;

Check the balances again to confirm the transaction succeeded:

SELECT * FROM bank_account;

This demonstrates atomicity: either the entire transaction is completed, or no changes are made at all, preserving database integrity.


Consistency

Consistency ensures that a database remains in a valid state before and after a transaction, meaning any transaction will take the database from one valid state to another. It guarantees that all predefined rules, such as constraints, are respected.

Setup:

To enforce consistency, we will add a CHECK constraint to ensure that the balance in any account can never be less than zero.

Modify the bank_account table to include this constraint:

ALTER TABLE bank_account
ADD CONSTRAINT positive_balance CHECK (balance >= 0);

This rule guarantees that no transaction can result in a negative balance for any account, ensuring the database’s consistency.


Testing Consistency:


Let’s try to transfer more money from Account A than its current balance to see how the consistency is maintained.

Start a transaction:

BEGIN;


Attempt to transfer $1000 from Account A (which only has $800):

UPDATE bank_account SET balance = balance - 1000 WHERE account_name = 'Account A';

UPDATE bank_account SET balance = balance + 1000 WHERE account_name = 'Account B';

Since Account A only has $800, the CHECK constraint will prevent this transaction from violating the rule:

ERROR:  new row for relation "bank_account" violates check constraint "positive_balance"
DETAIL:  Failing row contains (1, Account A, -200).
This error occurs because the operation would have resulted in a negative balance, which violates the consistency rule we set.

Successful Transaction:
Now, let’s try a valid transfer that maintains consistency:

BEGIN;
UPDATE bank_account SET balance = balance - 200 WHERE account_name = 'Account A';
UPDATE bank_account SET balance = balance + 200 WHERE 
account_name = 'Account B';
COMMIT;
This transaction follows the constraint that no account can have a negative balance, so it completes successfully. The balances are updated correctly without violating the consistency of the database.


Isolation Example
Let’s run two simultaneous transactions that attempt to update the same row. Without proper isolation, these could interfere with each other.

Open two separate PostgreSQL sessions (using the same docker exec command). Run the following commands in each:

Session 1:

postgres=# BEGIN;

-- Select balance of Account A
SELECT * FROM bank_account WHERE account_name = 'Account A';

-- Deduct $100 from Account A
UPDATE bank_account SET balance = balance - 100 WHERE account_name = 'Account A';
BEGIN
 id | account_name | balance 
----+--------------+---------
  1 | Account A    |    1000
(1 row)



-- Do not commit yet; leave the transaction open
Session 2:

postgres=# BEGIN;
BEGIN
postgres=*# SELECT * FROM bank_account WHERE account_name = 'Account A';
 id | account_name | balance 
----+--------------+---------
  1 | Account A    |    1000
(1 row)

postgres=*# UPDATE bank_account SET balance = balance - 200 WHERE account_name = 'Account A';

COMMIT;
You’ll see that Session 2 is blocked until Session 1 completes, demonstrating the isolation property. PostgreSQL uses isolation to prevent inconsistent results due to concurrent transactions.