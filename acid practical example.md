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