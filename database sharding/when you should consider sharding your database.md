
-- Considerations for when to shard a database:

-- 1. Data volume exceeds single server capacity
-- When total data size approaches storage limits
-- When query performance degrades due to large table sizes

-- 2. Write throughput bottlenecks 
-- When write operations exceed single server capacity
-- When write latency increases beyond acceptable thresholds

-- 3. Read throughput limitations
-- When read operations saturate server resources
-- When read latency increases due to high concurrency

-- 4. Geographic distribution needs
-- When data needs to be closer to users in different regions
-- When regulatory requirements mandate data locality

-- 5. Resource isolation requirements
-- When certain customers/tenants need dedicated resources
-- When workload isolation is required for performance

-- 6. Cost optimization
-- When vertical scaling becomes cost-prohibitive
-- When resource utilization is uneven across data

-- Key metrics indicating need for sharding:
SELECT 
    'Database Size (GB)' as metric,
    pg_database_size(current_database())/1024/1024/1024 as value
UNION ALL
SELECT 
    'Table Count',
    count(*) 
FROM information_schema.tables
UNION ALL
SELECT
    'Active Connections',
    count(*) 
FROM pg_stat_activity
UNION ALL
SELECT
    'Write Operations/sec',
    sum(tup_inserted + tup_updated + tup_deleted)
FROM pg_stat_database
WHERE datname = current_database();

-- Sharding considerations checklist:
/*
□ Data size > 1TB
□ Write operations > 10k/sec
□ Read operations > 100k/sec
□ Geographic latency > 100ms
□ Cost of vertical scaling > 2x current
□ Regulatory requirements for data locality
□ Need for tenant isolation
□ Uneven resource utilization
*/