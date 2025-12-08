# PostgreSQL Replication

## Overview
Replication creates copies of your database on multiple servers for high availability, load balancing, and disaster recovery.

## Types of Replication

### 1. Streaming Replication (Physical)
- Replicates entire database cluster at binary level
- Standby servers are exact copies
- Read-only queries on standby (hot standby)

### 2. Logical Replication
- Replicates specific tables/databases
- Allows writes on subscriber
- More flexible but slower

---

## Streaming Replication Setup

### Primary Server Configuration

**1. Edit postgresql.conf:**
```conf
wal_level = replica
max_wal_senders = 3
wal_keep_size = 64
```

**2. Edit pg_hba.conf:**
```conf
host replication replicator 192.168.1.0/24 md5
```

**3. Create replication user:**
```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'password';
```

**4. Restart PostgreSQL:**
```bash
sudo systemctl restart postgresql
```

### Standby Server Configuration

**1. Stop PostgreSQL:**
```bash
sudo systemctl stop postgresql
```

**2. Remove data directory:**
```bash
rm -rf /var/lib/postgresql/14/main/*
```

**3. Base backup from primary:**
```bash
pg_basebackup -h primary_host -D /var/lib/postgresql/14/main -U replicator -P -v -R -X stream -C -S standby1
```

**4. Start PostgreSQL:**
```bash
sudo systemctl start postgresql
```

---

## Logical Replication Setup

### Publisher (Source) Configuration

**1. Edit postgresql.conf:**
```conf
wal_level = logical
```

**2. Create publication:**
```sql
CREATE PUBLICATION my_publication FOR TABLE users, orders;
-- OR for all tables
CREATE PUBLICATION all_tables FOR ALL TABLES;
```

### Subscriber (Target) Configuration

**1. Create tables with same schema:**
```sql
CREATE TABLE users (...);
CREATE TABLE orders (...);
```

**2. Create subscription:**
```sql
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=primary_host dbname=mydb user=replicator password=password'
PUBLICATION my_publication;
```

---

## Monitoring Replication

### Check replication status (Primary):
```sql
SELECT * FROM pg_stat_replication;
```

### Check replication lag (Standby):
```sql
SELECT 
    now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

### Check WAL sender processes:
```sql
SELECT pid, usename, application_name, client_addr, state, sync_state
FROM pg_stat_replication;
```

---

## Failover & Promotion

### Promote standby to primary:
```bash
pg_ctl promote -D /var/lib/postgresql/14/main
```

### Or using SQL:
```sql
SELECT pg_promote();
```

---

## Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| wal_level | Amount of WAL info | replica |
| max_wal_senders | Max concurrent connections | 10 |
| wal_keep_size | Min WAL size to keep | 0 |
| hot_standby | Allow queries on standby | on |
| max_replication_slots | Max replication slots | 10 |

---

## Best Practices

1. **Use replication slots** - Prevents WAL deletion before standby reads it
2. **Monitor lag** - Set up alerts for replication delays
3. **Test failover** - Regularly practice promotion procedures
4. **Secure connections** - Use SSL for replication traffic
5. **Backup strategy** - Replication ≠ backup, maintain separate backups

---

## Common Issues

**Replication lag:**
- Network bandwidth issues
- Heavy write load on primary
- Slow disk I/O on standby

**Connection failures:**
- Check pg_hba.conf settings
- Verify network connectivity
- Confirm replication user permissions

**WAL files accumulating:**
- Standby not consuming WAL fast enough
- Use replication slots or adjust wal_keep_size
