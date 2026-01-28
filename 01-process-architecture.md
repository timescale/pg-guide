# Process Architecture

## What is it?

PostgreSQL uses a **multi-process architecture** (not multi-threaded) where each client connection spawns a separate operating system process called a **backend process**. Additionally, PostgreSQL runs several **auxiliary processes** that handle background tasks essential for database operation.

### Key Components

#### Backend Processes (postgres)
- One process per client connection
- Handles all SQL queries for that connection
- Has its own memory space (separate from other backends)
- Dies when the client disconnects

#### Postmaster (Main Process)
- The parent process that starts when PostgreSQL starts
- Listens for incoming connections
- Spawns new backend processes for each connection
- Manages auxiliary processes

#### Auxiliary Processes
- **WAL Writer**: Flushes Write-Ahead Log buffers to disk
- **Background Writer**: Writes dirty buffers from shared memory to disk
- **Checkpointer**: Performs checkpoints (coordinated flush of dirty pages)
- **Autovacuum Launcher/Workers**: Automatically runs VACUUM and ANALYZE
- **Stats Collector**: Gathers database statistics
- **Logical Replication Workers**: Handle logical replication subscriptions (if used)
- **Archiver**: Archives WAL files (if archiving is enabled)

## Why it matters

### Memory Implications
- Each backend process has its own **private memory** (work_mem, temp buffers, etc.)
- 1000 connections = 1000 separate processes
- Memory overhead can be significant: each process uses ~5-10MB base + query memory
- **CRITICAL**: A single query can use **multiple work_mem buffers** (1...N)
  - Example: Query with sort + hash join = 2× work_mem
  - Complex query with multiple sorts/hashes = N× work_mem per query
  - Parallel queries multiply this further (each worker gets work_mem)
- Real risk: work_mem × nodes_per_query × parallel_workers × connections
- Example: work_mem=64MB, 3 operations, 2 parallel workers, 100 connections = **38.4GB potential usage**
- **This memory usage can lead to OOM (Out of Memory) kills**, especially in containerized environments with memory limits

### Connection Limits
- PostgreSQL has a hard limit on connections (`max_connections`, default 100)
- **Changing max_connections requires a full server restart** - you can't adjust it on the fly
- This makes it impractical to frequently change; plan capacity carefully upfront
- Creating new processes is expensive (fork() overhead)
- Too many connections degrades performance (context switching, memory pressure)

### Why Connection Pooling is Critical
Since each connection = a process:
- Connection poolers like **[PgBouncer](https://www.pgbouncer.org)**, **[pgcat](https://github.com/postgresml/pgcat)**, or **[pgdog](https://pgdog.dev)** sit between clients and PostgreSQL
- They multiplex many client connections to a smaller pool of database connections
- Dramatically reduces process count and memory usage
- Typical setup: 1000 app connections → Connection Pooler → 20-50 PostgreSQL backends

### Process Isolation Benefits
- A crashed backend doesn't crash other connections
- Easier to identify resource-hogging queries (can see in OS process list)
- Simpler debugging with standard OS tools (gdb, strace, etc.)

## How to monitor

> [!TIP]
> For a visual, interactive alternative to the SQL queries below, check out **[pg_activity](https://github.com/dalibo/pg_activity)** - a top-like monitoring tool that shows real-time connection and query activity in your terminal.

### View Active Connections
Shows all non-idle backend processes with their queries. Useful for identifying what's currently running.

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    backend_start,
    backend_type,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY backend_start;
```

**Example output:**
```
  pid  | usename  | application_name | client_addr |      backend_start       |    backend_type    |        state        | query
-------+----------+------------------+-------------+--------------------------+--------------------+---------------------+------------------
 12345 | app_user | myapp            | 10.0.1.42   | 2026-01-20 10:23:45.123 | client backend     | active              | SELECT * FROM...
 12346 | app_user | myapp            | 10.0.1.43   | 2026-01-20 10:24:12.456 | client backend     | idle in transaction | BEGIN;
 12347 | postgres | psql             | 127.0.0.1   | 2026-01-20 10:25:01.789 | client backend     | active              | VACUUM users;
 12348 |          |                  |             | 2026-01-20 10:00:12.345 | autovacuum worker  | active              | autovacuum: ...
```

**What to look for:**
- `backend_type = 'client backend'`: Normal client connections - **this is what you typically care about**
- Other backend_types: `autovacuum worker`, `logical replication worker`, `background worker`, etc.
- `state = 'active'`: Currently executing a query
- `state = 'idle in transaction'`: Transaction open but not doing anything (can block VACUUM!)
- Long-running queries: Check `now() - query_start` to find queries running for too long

### Count Connections by State
Quick overview of connection states. Helps identify connection pool issues or stuck transactions.

```sql
SELECT
    state,
    COUNT(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count DESC;
```

**Example output:**
```
        state        | count
---------------------+-------
 idle                |   145
 active              |    12
 idle in transaction |     3
```

**What to look for:**
- High number of `idle in transaction`: Possible connection leak or application not committing/rolling back
- High number of `active`: Heavy load or slow queries
- Total count approaching `max_connections`: Need connection pooling or increase limit

### Check Connection Limit Usage
Shows how close you are to hitting `max_connections`. Critical for capacity planning.

```sql
SELECT
    COUNT(*) as current_connections,
    current_setting('max_connections')::int as max_connections,
    current_setting('superuser_reserved_connections')::int as superuser_reserved,
    current_setting('max_connections')::int -
      current_setting('superuser_reserved_connections')::int as non_superuser_limit,
    ROUND(100.0 * COUNT(*) / current_setting('max_connections')::int, 2) as pct_used
FROM pg_stat_activity;
```

**Example output:**
```
 current_connections | max_connections | superuser_reserved | non_superuser_limit | pct_used
---------------------+-----------------+--------------------+---------------------+----------
                 167 |             200 |                  3 |                 197 |    83.50
```

> [!IMPORTANT]
> `superuser_reserved_connections` (default: 3) are reserved for superuser emergency access. Normal users will get "too many clients" error when reaching `non_superuser_limit` (197 in example above). This reservation ensures DBAs can always connect to fix issues, even when the database is "full".

**What to look for:**
- `pct_used > 80%`: Time to implement connection pooling or increase max_connections
- `pct_used > 95%`: Critical - new connections will fail soon for non-superusers
- Consistent high usage: May indicate need for connection pooling, or simply high legitimate load

### View Backend Processes (OS Level)
See all PostgreSQL processes from the operating system perspective. Useful for identifying resource usage.

```bash
# Linux/Mac - see all postgres processes
ps aux | grep postgres

# See process tree structure
pstree -p $(pgrep -o postgres)
```

**Example output (ps aux):**
```
USER       PID  %CPU %MEM    VSZ   RSS STAT STARTED      TIME COMMAND
postgres  1234  0.0  2.1 345678 87654 Ss   10:00     0:00 /usr/bin/postgres -D /var/lib/postgresql/data
postgres  1235  0.0  0.5 345890 12345 Ss   10:00     0:01 postgres: checkpointer
postgres  1236  0.0  0.3 345890  9876 Ss   10:00     0:00 postgres: background writer
postgres  1237  0.0  0.4 345890 11234 Ss   10:00     0:02 postgres: walwriter
postgres  1238  0.0  0.5 346012 13456 Ss   10:00     0:00 postgres: autovacuum launcher
postgres  1239  0.0  0.3 345890  8765 Ss   10:00     0:00 postgres: logical replication launcher
postgres  5432  2.3  1.2 356789 34567 Ss   10:23     1:45 postgres: app_user mydb 10.0.1.42(54321) SELECT
postgres  5433  0.1  0.8 347890 23456 Ss   10:24     0:12 postgres: app_user mydb 10.0.1.43(54322) idle
```

**What to look for:**
- High `%CPU`: Identify which backend/query is consuming CPU
- High `RSS` (memory): Backends using excessive memory
- Auxiliary processes consuming unusual resources

**Example output (pstree):**
```
postgres(1234)─┬─postgres(1235) checkpointer
               ├─postgres(1236) background writer
               ├─postgres(1237) walwriter
               ├─postgres(1238) autovacuum launcher
               ├─postgres(1239) logical replication launcher
               ├─postgres(5432) app_user SELECT
               └─postgres(5433) app_user idle
```

### Monitor Auxiliary Processes
Focus specifically on PostgreSQL's background workers.

```bash
# Look for specific auxiliary processes
ps aux | grep -E "postgres.*(wal writer|checkpointer|autovacuum|stats collector)"
```

**Example output:**
```
postgres  1235  0.0  0.5 345890 12345 Ss   10:00     0:01 postgres: checkpointer
postgres  1237  0.0  0.4 345890 11234 Ss   10:00     0:02 postgres: walwriter
postgres  1238  0.0  0.5 346012 13456 Ss   10:00     0:00 postgres: autovacuum launcher
postgres  8765  1.5  2.1 389012 56789 Ds   14:32     0:45 postgres: autovacuum worker process   database_name
```

**What to look for:**
- Multiple autovacuum workers: May indicate bloat issues or aggressive autovacuum settings
- High CPU on checkpointer: Checkpoint tuning may be needed (see WAL section)
- Missing processes: Critical failure if core auxiliary processes aren't running

## Common problems

### Problem: "FATAL: sorry, too many clients already"
**Symptom**: Applications can't connect, error about max_connections

**Solutions**:
1. Increase `max_connections` (requires restart)
2. **Better**: Implement connection pooling (PgBouncer)
3. Find and kill idle/stuck connections
4. Investigate connection leaks in application code

**Quick fix to find long-idle connections**:
```sql
SELECT
    pid,
    usename,
    state,
    state_change,
    now() - state_change as idle_duration,
    query
FROM pg_stat_activity
WHERE state = 'idle'
  AND now() - state_change > interval '1 hour'
ORDER BY state_change;

-- Terminate if needed (be careful!)
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE pid = <problem_pid>;
```

### Problem: High memory usage
**Symptom**: Server running out of memory, OOM killer triggered

**Causes**:
- Too many connections × work_mem is too large
- Long-running queries with large sorts/hashes
- No connection pooling

**Solutions**:
1. Reduce `work_mem` per connection
2. Implement connection pooling to reduce connection count
3. Set connection limits at application level
4. Use `statement_timeout` to kill runaway queries

### Problem: Process not responding
**Symptom**: A specific query is stuck, process consuming 100% CPU

**Investigation**:
```sql
-- Find the problematic query
SELECT pid, now() - query_start as duration, state, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- On OS level, see what it's doing
strace -p <pid>  # Linux
```

**Action**:
```sql
-- Try to cancel the query (graceful)
SELECT pg_cancel_backend(<pid>);

-- If cancel doesn't work, terminate (connection drops)
SELECT pg_terminate_backend(<pid>);
```

### Problem: Too many autovacuum workers
**Symptom**: Many autovacuum processes running, competing for resources

**Check**:
```sql
SELECT COUNT(*) FROM pg_stat_activity WHERE query LIKE 'autovacuum:%';
```

**Solution**: Tune `autovacuum_max_workers` (default 3) or adjust autovacuum thresholds.

## References

1. [PostgreSQL Documentation: Server Processes](https://www.postgresql.org/docs/current/tutorial-arch.html)
2. [PostgreSQL Documentation: Connection Management](https://www.postgresql.org/docs/current/runtime-config-connection.html)
3. [PostgreSQL Documentation: Monitoring Database Activity](https://www.postgresql.org/docs/current/monitoring.html)
4. [The Internals of PostgreSQL: Process and Memory Architecture](https://www.interdb.jp/pg/pgsql02/01.html)
