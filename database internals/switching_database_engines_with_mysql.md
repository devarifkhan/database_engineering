# Switching Database Engines with MySQL

## MySQL Storage Engines

### Common Engines

| Engine | Transactions | Locking | Use Case |
|--------|-------------|---------|----------|
| InnoDB | Yes | Row-level | Default, ACID compliant |
| MyISAM | No | Table-level | Read-heavy, full-text search |
| Memory | No | Table-level | Temporary data, caching |
| Archive | No | Row-level | Historical data, compression |

---

## Check Current Engine

```sql
-- Check table engine
SHOW TABLE STATUS WHERE Name = 'users';

-- Or
SELECT ENGINE FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'users';

-- List all tables with engines
SELECT TABLE_NAME, ENGINE 
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'mydb';
```

---

## Switch Engine

### Single Table
```sql
ALTER TABLE users ENGINE = InnoDB;
```

### Multiple Tables
```sql
ALTER TABLE users ENGINE = InnoDB;
ALTER TABLE orders ENGINE = InnoDB;
ALTER TABLE products ENGINE = InnoDB;
```

### All Tables in Database
```sql
-- Generate ALTER statements
SELECT CONCAT('ALTER TABLE ', TABLE_NAME, ' ENGINE=InnoDB;') 
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'mydb' AND ENGINE != 'InnoDB';
```

---

## Engine Comparison

### InnoDB (Default)
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
) ENGINE=InnoDB;
```
**Features:**
- ACID transactions
- Foreign keys
- Row-level locking
- Crash recovery

### MyISAM
```sql
CREATE TABLE logs (
    id INT PRIMARY KEY,
    message TEXT
) ENGINE=MyISAM;
```
**Features:**
- Fast reads
- Full-text search (pre-5.6)
- Table-level locking
- No transactions

### Memory
```sql
CREATE TABLE session_data (
    session_id VARCHAR(32) PRIMARY KEY,
    data TEXT
) ENGINE=Memory;
```
**Features:**
- Stored in RAM
- Very fast
- Data lost on restart

### Archive
```sql
CREATE TABLE audit_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    action VARCHAR(100),
    created_at TIMESTAMP
) ENGINE=Archive;
```
**Features:**
- High compression
- Insert and select only
- No indexes except AUTO_INCREMENT

---

## Migration Considerations

### Before Switching

**1. Check foreign keys (InnoDB only):**
```sql
SELECT * FROM information_schema.KEY_COLUMN_USAGE 
WHERE TABLE_SCHEMA = 'mydb' AND REFERENCED_TABLE_NAME IS NOT NULL;
```

**2. Check table size:**
```sql
SELECT TABLE_NAME, 
       ROUND(((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024), 2) AS 'Size (MB)'
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'mydb';
```

**3. Backup:**
```bash
mysqldump -u root -p mydb > backup.sql
```

### During Migration

**Large tables - use online DDL:**
```sql
ALTER TABLE large_table ENGINE=InnoDB, ALGORITHM=INPLACE, LOCK=NONE;
```

**Monitor progress:**
```sql
SHOW PROCESSLIST;
```

---

## Performance Impact

### MyISAM to InnoDB
- **Pros:** ACID, better concurrency, crash recovery
- **Cons:** More disk space, slightly slower reads
- **Downtime:** Table locked during conversion

### InnoDB to MyISAM
- **Pros:** Faster reads, less disk space
- **Cons:** No transactions, table-level locks
- **Risk:** Data integrity issues

---

## Best Practices

1. **Use InnoDB by default** - Modern standard
2. **Test on staging** - Verify performance before production
3. **Schedule maintenance window** - Large tables take time
4. **Monitor disk space** - InnoDB uses more space
5. **Backup first** - Always have rollback plan

---

## Common Issues

**Foreign key constraints:**
```sql
-- Disable checks temporarily
SET FOREIGN_KEY_CHECKS=0;
ALTER TABLE users ENGINE=InnoDB;
SET FOREIGN_KEY_CHECKS=1;
```

**Out of disk space:**
```sql
-- Check available space
SELECT @@datadir;
```

**Long conversion time:**
```sql
-- Use pt-online-schema-change (Percona Toolkit)
pt-online-schema-change --alter "ENGINE=InnoDB" D=mydb,t=users --execute
```

---

## Set Default Engine

```sql
-- Session level
SET default_storage_engine=InnoDB;

-- Global level
SET GLOBAL default_storage_engine=InnoDB;
```

**In my.cnf:**
```ini
[mysqld]
default-storage-engine=InnoDB
```
