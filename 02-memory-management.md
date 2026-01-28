# Memory Management

## What is it?

PostgreSQL uses a sophisticated memory management system with **both shared and private memory areas**. Understanding how PostgreSQL allocates and uses memory is critical for performance tuning and avoiding out-of-memory (OOM) situations.

### Key Memory Areas

#### Shared Memory (shared across all processes)

**shared_buffers**
- PostgreSQL's main data cache
- Stores frequently accessed table and index pages
- Shared by all backend processes
- Set once at server start (requires restart to change)
- Sizing depends on workload (OLTP, OLAP, mixed) and total system memory
- Balance between shared_buffers and OS page cache is workload-dependent
- Use tools like **[PGConfig.io](https://pgconfig.io)** for initial sizing recommendations
- Fine-tune based on cache hit ratios and workload testing

#### Private Memory (per backend process)

**work_mem**
- Used for sorting, hash tables, and merge joins
- **Defines the maximum memory** per operation - actual usage can be less
- Operations allocate memory on-demand up to the work_mem limit
- If data exceeds work_mem, PostgreSQL spills to temporary disk files (slow!)
- **Each backend can use multiple work_mem allocations** (one per operation)
- **There is NO hard limit** on allocations per query - complex queries can use 10+ work_mem buffers
- Hash operations use `hash_mem_multiplier` (default: 2.0) × work_mem
- Parallel workers each get their own work_mem allocation
- **Critical**: Most dangerous memory parameter - see "Memory Multiplication Danger" section below

**maintenance_work_mem**
- Used for maintenance operations: VACUUM, CREATE INDEX, ALTER TABLE
- Can be set much higher than work_mem since fewer processes use it
- Typical size: 1-2GB (or higher for large databases)

**temp_buffers**
- Used for temporary tables within a session
- Default: 8MB per session
- Rarely needs adjustment

#### Planning Parameters

**effective_cache_size**
- **Not an allocation** - this is a hint to the query planner
- Tells the planner how much memory is available for caching (including OS cache)
- Should be set to ~50-75% of total system RAM
- Does NOT reserve or allocate memory

## Why it matters

### Performance Impact

**Insufficient shared_buffers**
- More disk I/O (cache misses)
- Slower query performance
- **Storage latency becomes a bottleneck** - every cache miss exposes queries to full storage latency (especially impactful with high-latency storage like cloud EBS, network-attached storage)

**Too large shared_buffers**
- Less memory available for OS page cache (kernel cache)
- PostgreSQL uses both shared_buffers and OS page cache - a miss in shared_buffers can still hit OS cache (avoiding disk I/O)
- The optimal split between shared_buffers and OS cache depends on workload (OLTP vs OLAP, working set size, etc.)
- Very large shared_buffers (>100GB) can slow PostgreSQL startup and make checkpoints heavier
- Use tools like **[PGConfig.io](https://pgconfig.io)** for workload-specific recommendations
- See [effective_cache_size docs](https://www.postgresql.org/docs/current/runtime-config-query.html#GUC-EFFECTIVE-CACHE-SIZE) which accounts for both kernel cache and shared_buffers

**Incorrect work_mem**
- Too small: Queries spill to disk (temp files), dramatically slower
- Too large: Risk of OOM kills, especially in containerized environments
- The multiplication effect makes this particularly dangerous

### Memory Multiplication Danger

The most common mistake is not accounting for the **maximum potential** memory usage:

```
Maximum potential memory = work_mem × operations_per_query × parallel_workers × connections
(Note: hash operations can use up to hash_mem_multiplier × work_mem, default 2.0)
```

> [!IMPORTANT]
> This is the **worst-case scenario**. Actual usage depends on data size - operations use memory on-demand **up to** their work_mem limit. However, planning for worst-case is critical to avoid OOM kills.

**Example (worst-case)**:
- work_mem = 128MB
- hash_mem_multiplier = 2.0 (default)
- Query has 3 operations (2 sorts + 1 hash join)
- 2 parallel workers per query
- 100 active connections

**Calculation (maximum potential)**:
- 2 sorts: **up to** 128MB × 2 = 256MB
- 1 hash join: **up to** 128MB × 2.0 (hash_mem_multiplier) = 256MB
- **Per query per worker**: up to 512MB
- **With 2 workers**: up to 512MB × 2 = 1024MB per query
- **With 100 connections**: up to 1024MB × 100 = **102.4GB**

In practice, not all queries will hit these limits simultaneously, but with high concurrency and large datasets, you can approach this worst-case scenario, easily triggering OOM kills in Kubernetes pods or cloud instances with memory limits.

**Key takeaway**: `hash_mem_multiplier` makes hash operations even more memory-hungry. Check this setting:
```sql
SHOW hash_mem_multiplier;  -- Default: 2.0
```

### Interaction with OS Cache

PostgreSQL's "double buffering" architecture:
1. **shared_buffers**: PostgreSQL's cache
2. **OS page cache**: Operating system's file cache

Data can exist in both caches, which is why extremely large shared_buffers isn't always beneficial. The OS cache is very efficient, and PostgreSQL relies on it.

## How to monitor

### Check Current Memory Settings

```sql
SELECT
    name,
    setting,
    unit,
    context
FROM pg_settings
WHERE name IN (
    'shared_buffers',
    'work_mem',
    'maintenance_work_mem',
    'effective_cache_size',
    'temp_buffers',
    'hash_mem_multiplier'
)
ORDER BY name;
```

**Example output:**
```
         name          | setting | unit  |  context
-----------------------+---------+-------+-----------
 effective_cache_size  | 524288  | 8kB   | user
 hash_mem_multiplier   | 2       |       | user
 maintenance_work_mem  | 262144  | kB    | user
 shared_buffers        | 131072  | 8kB   | postmaster
 temp_buffers          | 1024    | 8kB   | user
 work_mem              | 4096    | kB    | user
```

**What to look for:**
- `hash_mem_multiplier`: Default is 2.0, meaning hash operations can use 2× work_mem
- This multiplier significantly increases memory usage for queries with hash joins or hash aggregates

**Converting to human-readable sizes:**
```sql
SELECT
    name,
    setting,
    unit,
    pg_size_pretty(
        CASE
            WHEN unit = '8kB' THEN setting::bigint * 8 * 1024
            WHEN unit = 'kB' THEN setting::bigint * 1024
            WHEN unit = 'B' THEN setting::bigint
            ELSE NULL
        END
    ) AS size
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'maintenance_work_mem', 'effective_cache_size');
```

**Example output:**
```
         name          | setting | unit |  size
-----------------------+---------+------+--------
 effective_cache_size  | 524288  | 8kB  | 4096 MB
 maintenance_work_mem  | 262144  | kB   | 256 MB
 shared_buffers        | 131072  | 8kB  | 1024 MB
 work_mem              | 4096    | kB   | 4096 kB
```

### Monitor shared_buffers Usage (Cache Hit Ratio)

High cache hit ratio = data served from memory instead of disk.

```sql
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    ROUND(100.0 * sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables;
```

**Example output:**
```
 heap_read | heap_hit | cache_hit_ratio
-----------+----------+-----------------
    45623  | 8234561  |           99.45
```

**What to look for:**
- `cache_hit_ratio > 99%`: Excellent - most data served from cache
- `cache_hit_ratio 95-99%`: Good, but might benefit from more shared_buffers
- `cache_hit_ratio < 95%`: Increase shared_buffers or investigate query patterns
- **Note**: Low ratio on a freshly started database is normal (cold cache)

### Detect Queries Using Temp Files (work_mem too small)

When work_mem is insufficient, PostgreSQL spills to disk (creates temp files). This is SLOW.

```sql
SELECT
    query,
    calls,
    total_exec_time,
    temp_blks_written,
    temp_blks_read,
    pg_size_pretty(temp_blks_written * 8192::bigint) as temp_space_used
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 10;
```

**Example output:**
```
                    query                     | calls | total_exec_time | temp_blks_written | temp_space_used
----------------------------------------------+-------+-----------------+-------------------+-----------------
 SELECT * FROM large_table ORDER BY created   |   142 |       45623.23  |          1245632  | 9.5 GB
 SELECT * FROM users u JOIN orders o ON ...   |    89 |       23451.12  |           456789  | 3.5 GB
```

**What to look for:**
- `temp_blks_written > 0`: Queries spilling to disk due to insufficient work_mem
- High `temp_blks_written` with slow `total_exec_time`: Strong candidate for work_mem increase (or query optimization)
- **WARNING**: Don't blindly increase work_mem globally - consider per-query tuning with `SET work_mem = '256MB'` for specific sessions

**Enable temp file logging** to catch these in real-time:
```sql
-- Log temp files larger than 10MB
ALTER SYSTEM SET log_temp_files = 10240;  -- in kB
SELECT pg_reload_conf();
```

### Monitor Current Memory Usage (OS Level)

```bash
# Linux: See PostgreSQL memory usage
ps aux --sort=-%mem | grep postgres | head -20

# Total PostgreSQL memory usage
ps -C postgres -o pid,user,vsz,rss,cmd --sort=-rss | head -20
```

**Example output:**
```
USER       PID %CPU %MEM    VSZ   RSS STAT STARTED      TIME COMMAND
postgres  1234  2.1  8.5 2456789 876543 Ss   10:00     1:23 /usr/bin/postgres -D /var/lib/postgresql/data
postgres  5432  1.2  3.2 2345678 345678 Ss   10:23     0:45 postgres: app_user mydb SELECT (sorting)
postgres  5433  0.8  2.1 2234567 234567 Ss   10:24     0:12 postgres: app_user mydb SELECT (hashing)
```

**What to look for:**
- `RSS` column: Actual physical memory usage per process
- Backends using significantly more RSS than typical: Likely consuming multiple work_mem buffers
- Sum of all RSS values approaching system limits: Risk of OOM

### Check for OOM Kills (Linux)

```bash
# Check system logs for OOM killer activity
dmesg -T | grep -i "killed process"
journalctl -k | grep -i "out of memory"

# Check PostgreSQL logs for unexpected process terminations
grep -i "unexpected" /var/log/postgresql/postgresql-*.log
```

## Common problems

### Problem: Database cache hit ratio is low (<95%)

**Symptom**: Queries are slower than expected, high disk I/O

**Diagnosis**:
```sql
-- Check cache hit ratio per table
SELECT
    schemaname,
    tablename,
    heap_blks_read,
    heap_blks_hit,
    ROUND(100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY cache_hit_ratio ASC, heap_blks_read DESC
LIMIT 10;
```

**Solutions**:
1. Increase `shared_buffers` (requires restart)
2. Check if database just started (cold cache takes time to warm)
3. Investigate if working set is larger than available memory
4. Consider table partitioning to reduce working set size

### Problem: Queries creating large temp files

**Symptom**: Slow query performance, disk I/O spikes, temp file warnings in logs

**Diagnosis**: See "Detect Queries Using Temp Files" query above

**Solutions**:
1. **Per-session tuning** (preferred):
   ```sql
   SET work_mem = '256MB';
   -- Run your query
   ```
2. Optimize the query (add indexes, rewrite joins)
3. Increase `work_mem` globally - **DANGEROUS**, calculate worst case first
4. Use connection pooling to limit number of concurrent queries

### Problem: Out of memory (OOM) kills

**Symptom**: PostgreSQL processes killed unexpectedly, "postgres: ... killed" in dmesg

**Causes**:
- `work_mem` set too high with many concurrent connections
- Parallel queries multiplying work_mem usage
- Memory leaks in extensions or functions
- Insufficient system memory for workload

**Investigation**:
```sql
-- Find queries with high parallel workers
SELECT
    query,
    calls,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
WHERE query LIKE '%parallel%' OR query LIKE '%Parallel%'
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Check current parallel settings
SHOW max_parallel_workers_per_gather;
SHOW max_worker_processes;
```

**Solutions**:
1. Reduce `work_mem` significantly
2. Lower `max_parallel_workers_per_gather`
3. Implement connection pooling
4. Set per-query memory limits: `SET work_mem = '64MB';`
5. Increase system memory or use larger instance/pod
6. Set memory limits in postgresql.conf conservatively

### Problem: shared_buffers change not taking effect

**Symptom**: Changed `shared_buffers` in postgresql.conf but value unchanged

**Cause**: `shared_buffers` requires a full server restart (context = 'postmaster')

**Solution**:
```bash
# Check current value
psql -c "SHOW shared_buffers;"

# Edit postgresql.conf
vim /var/lib/postgresql/data/postgresql.conf

# Reload won't work - need full restart
sudo systemctl restart postgresql

# Or in Kubernetes
kubectl rollout restart statefulset postgres
```

### Problem: Maintenance operations (VACUUM, CREATE INDEX) running slowly

**Symptom**: VACUUM or index creation taking hours on large tables

**Diagnosis**:
```sql
SHOW maintenance_work_mem;

-- Check if autovacuum is competing for resources
SELECT COUNT(*) FROM pg_stat_activity WHERE query LIKE 'autovacuum:%';
```

**Solutions**:
1. Increase `maintenance_work_mem` (can be set per-session):
   ```sql
   SET maintenance_work_mem = '2GB';
   CREATE INDEX CONCURRENTLY idx_name ON table_name(column);
   ```
2. Run maintenance during off-peak hours
3. Use `CREATE INDEX CONCURRENTLY` to avoid blocking writes
4. Consider partitioning very large tables

## References

1. [PostgreSQL Documentation: Resource Consumption](https://www.postgresql.org/docs/current/runtime-config-resource.html)
2. [The Internals of PostgreSQL: Memory Architecture](https://www.interdb.jp/pg/pgsql02.html)
