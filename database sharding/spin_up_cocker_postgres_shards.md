# Database Sharding with PostgreSQL

## Build the Docker Image

```bash
docker build -t pgshard .
```

## Run the Shards

```bash
docker run --name pgshard1 -e POSTGRES_PASSWORD=postgres -p 5432:5432 -d pgshard
docker run --name pgshard2 -e POSTGRES_PASSWORD=postgres -p 5433:5432 -d pgshard
docker run --name pgshard3 -e POSTGRES_PASSWORD=postgres -p 5434:5432 -d pgshard
```

## Verify Containers are Running

```bash
docker ps
```

## Connect to Shards

```bash
psql -h localhost -p 5432 -U postgres  # pgshard1
psql -h localhost -p 5433 -U postgres  # pgshard2
psql -h localhost -p 5434 -U postgres  # pgshard3
```

Password: `postgres`

## Stop and Remove Containers

```bash
docker stop pgshard1 pgshard2 pgshard3
docker rm pgshard1 pgshard2 pgshard3
```
