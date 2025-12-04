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
