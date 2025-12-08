# How to Automate Partitioning in PostgreSQL

## Overview
Automating partition creation and management prevents manual intervention and ensures partitions are created before data arrives.

## Methods for Automation

### 1. Using pg_partman Extension

**Installation:**
```sql
CREATE EXTENSION pg_partman;
```

**Setup for Time-Based Partitioning:**
```sql
-- Create parent table
CREATE TABLE sales (
    id BIGSERIAL,
    sale_date TIMESTAMP NOT NULL,
    amount DECIMAL(10,2)
) PARTITION BY RANGE (sale_date);

-- Configure pg_partman
SELECT partman.create_parent(
    p_parent_table => 'public.sales',
    p_control => 'sale_date',
    p_type => 'native',
    p_interval => 'daily',
    p_premake => 7  -- Create 7 partitions in advance
);

-- Enable automatic maintenance
UPDATE partman.part_config 
SET infinite_time_partitions = true,
    retention = '90 days',
    retention_keep_table = false
WHERE parent_table = 'public.sales';
```

**Schedule Maintenance (run via cron):**
```sql
-- Run this periodically (e.g., daily via cron)
SELECT partman.run_maintenance_proc();
```

### 2. Using PL/pgSQL Function

**Create Automation Function:**
```sql
CREATE OR REPLACE FUNCTION create_monthly_partitions(
    table_name TEXT,
    months_ahead INTEGER DEFAULT 3
)
RETURNS VOID AS $$
DECLARE
    start_date DATE;
    end_date DATE;
    partition_name TEXT;
    i INTEGER;
BEGIN
    FOR i IN 0..months_ahead LOOP
        start_date := DATE_TRUNC('month', CURRENT_DATE + (i || ' months')::INTERVAL);
        end_date := start_date + INTERVAL '1 month';
        partition_name := table_name || '_' || TO_CHAR(start_date, 'YYYY_MM');
        
        -- Check if partition exists
        IF NOT EXISTS (
            SELECT 1 FROM pg_class WHERE relname = partition_name
        ) THEN
            EXECUTE format(
                'CREATE TABLE IF NOT EXISTS %I PARTITION OF %I 
                FOR VALUES FROM (%L) TO (%L)',
                partition_name, table_name, start_date, end_date
            );
            
            RAISE NOTICE 'Created partition: %', partition_name;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

**Usage:**
```sql
-- Create partitions for next 3 months
SELECT create_monthly_partitions('sales', 3);
```

### 3. Using Cron Job with psql

**Create Shell Script (create_partitions.sh):**
```bash
#!/bin/bash
PGPASSWORD=your_password psql -h localhost -U postgres -d mydb -c "
SELECT create_monthly_partitions('sales', 3);
"
```

**Add to Crontab:**
```bash
# Run daily at 2 AM
0 2 * * * /path/to/create_partitions.sh
```

### 4. Using pg_cron Extension

**Installation:**
```sql
CREATE EXTENSION pg_cron;
```

**Schedule Automatic Partition Creation:**
```sql
-- Run daily at 3 AM
SELECT cron.schedule(
    'create-partitions',
    '0 3 * * *',
    $$SELECT create_monthly_partitions('sales', 3)$$
);

-- View scheduled jobs
SELECT * FROM cron.job;
```

## Complete Example: Automated Monthly Partitioning

```sql
-- 1. Create partitioned table
CREATE TABLE orders (
    order_id BIGSERIAL,
    order_date TIMESTAMP NOT NULL,
    customer_id INTEGER,
    total DECIMAL(10,2)
) PARTITION BY RANGE (order_date);

-- 2. Create automation function
CREATE OR REPLACE FUNCTION maintain_order_partitions()
RETURNS VOID AS $$
DECLARE
    start_date DATE;
    end_date DATE;
    partition_name TEXT;
    i INTEGER;
BEGIN
    -- Create future partitions
    FOR i IN 0..2 LOOP
        start_date := DATE_TRUNC('month', CURRENT_DATE + (i || ' months')::INTERVAL);
        end_date := start_date + INTERVAL '1 month';
        partition_name := 'orders_' || TO_CHAR(start_date, 'YYYY_MM');
        
        IF NOT EXISTS (SELECT 1 FROM pg_class WHERE relname = partition_name) THEN
            EXECUTE format(
                'CREATE TABLE %I PARTITION OF orders 
                FOR VALUES FROM (%L) TO (%L)',
                partition_name, start_date, end_date
            );
            
            -- Create index on partition
            EXECUTE format(
                'CREATE INDEX %I ON %I (order_date)',
                partition_name || '_date_idx', partition_name
            );
        END IF;
    END LOOP;
    
    -- Drop old partitions (older than 6 months)
    FOR partition_name IN 
        SELECT tablename FROM pg_tables 
        WHERE tablename LIKE 'orders_%'
        AND TO_DATE(SUBSTRING(tablename FROM 8), 'YYYY_MM') < CURRENT_DATE - INTERVAL '6 months'
    LOOP
        EXECUTE format('DROP TABLE IF EXISTS %I', partition_name);
        RAISE NOTICE 'Dropped old partition: %', partition_name;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- 3. Schedule with pg_cron
SELECT cron.schedule(
    'maintain-orders',
    '0 2 * * *',
    $$SELECT maintain_order_partitions()$$
);
```

## Best Practices

1. **Pre-create Partitions**: Always create partitions ahead of time (1-3 months for monthly, 7-14 days for daily)
2. **Monitor Partition Creation**: Log partition creation events
3. **Automate Cleanup**: Remove old partitions based on retention policy
4. **Index Management**: Automatically create indexes on new partitions
5. **Test Automation**: Verify automation works before production deployment
6. **Handle Failures**: Add error handling and notifications
7. **Default Partition**: Create a default partition to catch unexpected data

## Monitoring

```sql
-- Check existing partitions
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE tablename LIKE 'orders_%'
ORDER BY tablename;

-- Check partition boundaries
SELECT 
    child.relname AS partition_name,
    pg_get_expr(child.relpartbound, child.oid) AS partition_bounds
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
WHERE parent.relname = 'orders';
```

## Troubleshooting

**Issue: Partition not created automatically**
- Check cron job is running
- Verify function has no errors
- Check PostgreSQL logs

**Issue: Data inserted into wrong partition**
- Verify partition boundaries don't overlap
- Check date/time values in data

**Issue: Performance degradation**
- Ensure indexes exist on partitions
- Check partition pruning is working (use EXPLAIN)
