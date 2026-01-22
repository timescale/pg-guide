# MVCC and Transactions

## What is it?

**MVCC (Multi-Version Concurrency Control)** is PostgreSQL's fundamental mechanism for handling concurrent transactions. Instead of using locks to manage concurrent access, PostgreSQL creates multiple versions of rows, allowing readers and writers to operate without blocking each other.

### Key Concepts

#### Row Versions (Tuples)

- Every UPDATE creates a **new version** of the row, keeping the old version
- DELETE marks a row as deleted but doesn't physically remove it immediately
- Each row version has visibility information (transaction IDs) determining who can see it
- Old versions become "dead tuples" after no transaction needs them

#### Transaction IDs (xid)

- Every transaction gets a unique transaction ID (32-bit integer)
- Transaction IDs are used to determine row visibility
- **xid wraparound**: After ~4 billion transactions, IDs wrap around (autovacuum prevents issues)

#### Visibility Rules

- Each transaction sees a consistent snapshot of the database
- Transactions only see rows committed before the transaction started (in READ COMMITTED) or before the snapshot was taken (in REPEATABLE READ/SERIALIZABLE)
- Concurrent transactions can modify different rows without blocking each other

**Visual Example: How MVCC Works**

```
┌────────────────────────────────┬──────────────────────────────────────┐
│       TRANSACTIONS             │      ROW VERSIONS (DISK)             │
├────────────────────────────────┼──────────────────────────────────────┤
│ Initial state                  │                                      │
│                                │  ┌────────────────────────────────┐  │
│                                │  │ v1: value=10                   │  │
│                                │  │     xmin=50, xmax=NULL         │  │
│                                │  │     (visible to all)           │  │
│                                │  └────────────────────────────────┘  │
├────────────────────────────────┼──────────────────────────────────────┤
│ [A] BEGIN (xid=100) ────┐      │                                      │
│     Snapshot: xmax=99   │      │  Same v1 above                       │
│                         │      │                                      │
│ [A] SELECT value ───────┼──────┼──> reads v1 (xmin=50 < 99)           │
│     returns 10          │      │      → returns 10                    │
│                         │      │                                      │
├─────────────────────────┼──────┼──────────────────────────────────────┤
│ [B] BEGIN (xid=101)     │      │                                      │
│                         │      │                                      │
│ [B] UPDATE value=20 ────┼──────┼──>┌────────────────────────────────┐ │
│                         │      │   │ v1: value=10                   │ │
│                         │      │   │     xmin=50, xmax=101   ◄──────┼─┼─ marked for deletion
│                         │      │   ├────────────────────────────────┤ │
│                         │      │   │ v2: value=20                   │ │
│                         │      │   │     xmin=101, xmax=NULL ◄──────┼─┼─ new version (not visible yet)
│                         │      │   └────────────────────────────────┘ │
│                         │      │                                      │
├─────────────────────────┼──────┼──────────────────────────────────────┤
│ [B] COMMIT              │      │                                      │
│                         │      │   v2 now visible to new snapshots    │
│                         │      │                                      │
├─────────────────────────┼──────┼──────────────────────────────────────┤
│ [A] SELECT value        │      │                                      │
│                         │      │                                      │
│  READ COMMITTED mode:   │      │                                      │
│    Takes NEW snapshot ──┼──────┼──> sees v2 (xmin=101 committed)      │
│    returns 20           │      │      → returns 20                    │
│                         │      │                                      │
│  REPEATABLE READ mode:  │      │                                      │
│    Uses OLD snapshot ───┼──────┼──> sees v1 (xmin=50, xmax=101>100)   │
│    (xmax=99)            │      │      → returns 10                    │
│                         │      │                                      │
├─────────────────────────┼──────┼──────────────────────────────────────┤
│ [A] COMMIT              │      │                                      │
│                         │      │                                      │
│                         │      │   v1 becomes DEAD TUPLE              │
│                         │      │   (needs VACUUM to reclaim space)    │
└─────────────────────────┴──────┴──────────────────────────────────────┘

Key concepts illustrated:
• xmin: transaction that created this row version
• xmax: transaction that deleted/updated this row version (NULL = current)
• Snapshot determines which xids are visible
• READ COMMITTED: new snapshot per statement
• REPEATABLE READ: one snapshot for entire transaction
• Dead tuples require VACUUM to reclaim space
```

#### Transaction Isolation Levels

PostgreSQL supports four isolation levels defined by the SQL standard:

**READ UNCOMMITTED** (treated as READ COMMITTED in PostgreSQL)
- PostgreSQL doesn't support true dirty reads
- Behaves identically to READ COMMITTED

**READ COMMITTED** (default)
- Each statement sees a fresh snapshot of committed data
- Can see different data within the same transaction
- Most common isolation level, good balance of consistency and performance

**REPEATABLE READ**
- Snapshot taken at start of first query in transaction
- All queries in the transaction see the same data
- Prevents non-repeatable reads and phantom reads
- Can cause serialization errors on write conflicts

**SERIALIZABLE**
- Strongest isolation level
- Transactions appear to execute serially
- Prevents all concurrency anomalies
- Can cause more serialization errors requiring retry logic

## Why it matters

### Performance Benefits

**Non-blocking reads**
- Readers never block writers
- Writers never block readers
- Only writers block other writers (on the same row)
- Enables high concurrency without lock contention

**No lock escalation**
- PostgreSQL doesn't escalate row locks to table locks
- Thousands of row locks don't degrade to table lock
- Predictable locking behavior

### Operational Impact

**VACUUM requirement**
- Dead tuples must be cleaned up by VACUUM
- Without VACUUM, tables bloat indefinitely
- **Performance impact of dead tuples**:
  - **Sequential scans**: Read blocks containing dead tuples from disk/cache, process them, then discard - wasted I/O and CPU
  - **Index scans**: Index entries can point to dead tuples, causing useless heap lookups
  - **Cache pollution**: Dead tuples occupy shared_buffers and OS page cache, reducing space for live data
  - **Larger physical size**: More disk blocks to read, slower backups, longer checkpoint times
  - **Index bloat**: Only when HOT (Heap-Only Tuple) updates don't apply - if UPDATEs modify indexed columns or page has no space, indexes grow with new entries. HOT updates avoid index bloat by keeping updates in the same heap page.

**Long-running transactions are dangerous**
- Block VACUUM from removing dead tuples
- Can cause severe table bloat
- Increase database size and slow down queries
- **Critical**: A single long-running transaction affects the ENTIRE database

**Transaction ID wraparound**
- Must vacuum regularly to prevent xid wraparound
- If wraparound occurs, database shuts down to prevent data loss
- Autovacuum has special aggressive mode to prevent this

### Concurrency Challenges

**UPDATE conflicts**
- Two transactions updating the same row: second one waits or fails
- In REPEATABLE READ/SERIALIZABLE: can cause serialization errors

**Serialization errors**
- Application must handle `ERROR: could not serialize access`
- Requires retry logic in application code
- More common in REPEATABLE READ and SERIALIZABLE

## How to monitor

### Check for Long-Running Transactions

Long-running transactions are the #1 cause of operational issues with MVCC.

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    now() - xact_start AS transaction_duration,
    now() - query_start AS query_duration,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start
LIMIT 10;
```

**Example output:**
```
  pid  | usename  | application_name |        state        | transaction_duration | query_duration |           query
-------+----------+------------------+---------------------+----------------------+----------------+---------------------------
 12345 | app_user | myapp            | idle in transaction | 02:15:32.456789      | 02:15:32.456789| BEGIN;
 12346 | app_user | myapp            | active              | 00:05:23.123456      | 00:00:02.345678| SELECT * FROM orders...
 12347 | etl_user | data_pipeline    | active              | 00:45:12.987654      | 00:45:12.987654| INSERT INTO events...
```

**What to look for:**
- `transaction_duration > 1 hour`: **Critical** - likely blocking VACUUM
- `state = 'idle in transaction'`: Transaction open but not doing anything - connection leak or forgotten BEGIN
- Long transactions from batch jobs/ETL: Should be broken into smaller transactions

### Monitor Dead Tuples and Bloat

Dead tuples accumulate when VACUUM can't clean them up (usually due to long transactions).

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_tup_pct,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 10;
```

**For detailed bloat analysis**, use community scripts:
- [pgx_scripts: table_bloat_check.sql](https://github.com/pgexperts/pgx_scripts/blob/master/bloat/table_bloat_check.sql) - More accurate bloat estimation
- [PostgreSQL Wiki: Show database bloat](https://wiki.postgresql.org/wiki/Show_database_bloat) - Multiple bloat queries for tables and indexes

**Example output:**
```
 schemaname | relname  | n_live_tup | n_dead_tup | dead_tup_pct |        last_vacuum        |      last_autovacuum
------------+----------+------------+------------+--------------+---------------------------+--------------------------
 public     | orders   |   5000000  |    450000  |         8.26 | 2026-01-20 10:00:00+00    | 2026-01-22 14:23:45+00
 public     | events   |  10000000  |    200000  |         1.96 |                           | 2026-01-22 15:00:12+00
 public     | users    |    500000  |    150000  |        23.08 | 2026-01-21 03:00:00+00    | 2026-01-22 15:30:00+00
```

**What to look for:**
- `dead_tup_pct > 10%`: Table needs vacuuming
- `dead_tup_pct > 20%`: **Concerning** - check for long transactions or autovacuum issues
- High `n_dead_tup` with recent `last_autovacuum`: Long transaction preventing cleanup
- No recent vacuum activity: Autovacuum may be disabled or overwhelmed

### Check Transaction ID Age (xid wraparound risk)

**⚠️ CRITICAL**: Transaction ID wraparound is one of the most severe operational issues in PostgreSQL. Understanding it is essential.

#### What is xid wraparound?

PostgreSQL uses 32-bit transaction IDs (xids), which wrap around after ~4 billion transactions (2^31). Without proper vacuuming, old transaction IDs can appear "in the future" due to wraparound, causing **data visibility corruption**.

#### The solution: VACUUM FREEZE

**What is VACUUM FREEZE?**

VACUUM FREEZE is a special operation that replaces old transaction IDs (xids) with a special value called **FrozenXID** (typically `2`). This special xid is treated as "always in the past" - meaning it's visible to all transactions, regardless of the current xid counter.

**How it works:**

1. **Normal row**: `xmin=3,999,999,995` - visible only to transactions with xid > 3,999,999,995
2. **After VACUUM FREEZE**: `xmin=2 (FrozenXID)` - visible to ALL transactions, forever

**When does it happen?**

- **Autovacuum** automatically freezes tuples older than `vacuum_freeze_min_age` (default: 50 million transactions)
- **Aggressive autovacuum** runs when database age reaches `autovacuum_freeze_max_age` (default: 200 million transactions)
- **Manual VACUUM FREEZE** can be run explicitly: `VACUUM FREEZE table_name;`

**Why it prevents wraparound:**

By replacing old xids with FrozenXID, rows become immune to the wraparound problem. After wraparound, when current xid resets to 3, frozen rows (xmin=2) are still considered "in the past" and remain visible.

**Visual example of wraparound corruption:**

```
┌────────────────────────────────┬──────────────────────────────────────┐
│      TRANSACTION STATE         │      TABLE DATA (users table)        │
├────────────────────────────────┼──────────────────────────────────────┤
│ BEFORE WRAPAROUND              │                                      │
│                                │                                      │
│ Current xid: 4,000,000,000     │  ┌────────────────────────────────┐  │
│                                │  │ Row 1: 'Alice'   xmin=3,999,995│  │
│ New transaction starts:        │  │ Row 2: 'Bob'     xmin=3,999,996│  │
│ → Gets xid=4,000,000,001       │  │ Row 3: 'Charlie' xmin=3,999,997│  │
│                                │  │ Row 4: 'Diana'   xmin=3,999,998│  │
│ SELECT * FROM users;           │  │ Row 5: 'Eve'     xmin=3,999,999│  │
│                                │  └────────────────────────────────┘  │
│ Visibility check:              │                                      │
│ xmin < current_xid?            │  ✓ All 5 rows visible                │
│ 3,999,995 < 4,000,000,001      │                                      │
│ → YES, visible                 │  Query returns: 5 rows               │
│                                │                                      │
├────────────────────────────────┼──────────────────────────────────────┤
│ 🔴 AFTER WRAPAROUND            │                                      │
│    (WITHOUT VACUUM)            │                                      │
│                                │                                      │
│ xid counter wraps:             │  ┌────────────────────────────────┐  │
│ 4,294,967,295 → 3              │  │ Row 1: 'Alice'   xmin=3,999,995│  │
│                                │  │ Row 2: 'Bob'     xmin=3,999,996│  │
│ Current xid: 3                 │  │ Row 3: 'Charlie' xmin=3,999,997│  │
│                                │  │ Row 4: 'Diana'   xmin=3,999,998│  │
│ New transaction starts:        │  │ Row 5: 'Eve'     xmin=3,999,999│  │
│ → Gets xid=4                   │  └────────────────────────────────┘  │
│                                │                                      │
│ SELECT * FROM users;           │  ✗ ALL ROWS INVISIBLE!               │
│                                │                                      │
│ Visibility check:              │  PostgreSQL logic:                   │
│ xmin < current_xid?            │  "xmin=3,999,995 is HUGE number!"    │
│ 3,999,995 < 4                  │  "Must be in the FUTURE!"            │
│ → NO! (wraparound!)            │                                      │
│                                │  Query returns: 0 rows               │
│ Result: DATA VANISHES          │  (Data physically exists on disk,    │
│                                │   but invisible to all queries)      │
│                                │                                      │
├────────────────────────────────┼──────────────────────────────────────┤
│ ✅ WITH VACUUM FREEZE          │                                      │
│    (PROPER OPERATION)          │                                      │
│                                │                                      │
│ Before wraparound,             │  ┌────────────────────────────────┐  │
│ VACUUM FREEZE runs:            │  │ Row 1: 'Alice'   xmin=2 (FROZ) │  │
│                                │  │ Row 2: 'Bob'     xmin=2 (FROZ) │  │
│ → Replaces old xids with       │  │ Row 3: 'Charlie' xmin=2 (FROZ) │  │
│   FrozenXID (special value=2)  │  │ Row 4: 'Diana'   xmin=2 (FROZ) │  │
│                                │  │ Row 5: 'Eve'     xmin=2 (FROZ) │  │
│ After wraparound:              │  └────────────────────────────────┘  │
│ Current xid: 4                 │                                      │
│                                │  ✓ All rows still visible!           │
│ SELECT * FROM users;           │                                      │
│                                │  FrozenXID is special:               │
│ Visibility check:              │  "xmin=2 (FrozenXID) is ALWAYS       │
│ FrozenXID=2 is ALWAYS visible  │   visible to ANY transaction"        │
│                                │                                      │
│                                │  Query returns: 5 rows               │
│                                │  (Immune to wraparound)              │
└────────────────────────────────┴──────────────────────────────────────┘

Key insight: Without VACUUM FREEZE, old xids become "future" xids after
wraparound, making ALL old data invisible. This looks like complete data
loss to the application, even though data physically exists on disk.
```

#### What actually happens during wraparound emergency?

**Stage 1: Warnings (starts at ~1.4 billion xids old)**
```
WARNING: database "mydb" must be vacuumed within 10000000 transactions
HINT: To avoid a database shutdown, execute a database-wide VACUUM in that database.
```

**Stage 2: Shutdown Protection (at ~2 billion xids old)**
```
ERROR: database is not accepting commands to avoid wraparound data loss in database "mydb"
HINT: Stop the postmaster and vacuum that database in single-user mode.
```

**At this point:**
- **Database becomes COMPLETELY INACCESSIBLE** to all applications
- Only superuser in single-user mode can connect
- **Your application is DOWN** until VACUUM completes
- VACUUM must process the **entire database** - can take **hours to days** on large databases
- No way to speed it up - it must scan every table

#### The real danger: Data visibility corruption

If PostgreSQL didn't shut down, the wraparound would cause:

1. **Old rows become invisible**: Rows with old xids appear as "in the future", becoming invisible to queries
2. **Data appears to vanish**: SELECT queries return incomplete results
3. **Logical corruption**: Data physically exists but is invisible - looks like data loss to application
4. **No recovery**: Once visibility is corrupted, you need PITR restore or data is permanently invisible

**PostgreSQL shuts down preemptively to prevent this catastrophic scenario.**

#### Monitoring query

```sql
SELECT
    datname,
    age(datfrozenxid) AS xid_age,
    2147483648 - age(datfrozenxid) AS xids_until_wraparound,
    ROUND(100.0 * age(datfrozenxid) / 2147483648, 2) AS pct_toward_wraparound,
    CASE
        WHEN age(datfrozenxid) > 2000000000 THEN '🔴 EMERGENCY - DB WILL SHUT DOWN'
        WHEN age(datfrozenxid) > 1500000000 THEN '🟠 CRITICAL - Immediate action required'
        WHEN age(datfrozenxid) > 1000000000 THEN '🟡 WARNING - Schedule aggressive vacuum'
        ELSE '🟢 OK'
    END AS status
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

**Example output:**
```
  datname  | xid_age    | xids_until_wraparound | pct_toward_wraparound | status
-----------+------------+-----------------------+-----------------------+----------------------------------
 myapp_db  | 1800000000 |             347483648 |                 83.82 | 🟠 CRITICAL - Immediate action
 postgres  |   10000000 |            2137483648 |                  0.47 | 🟢 OK
 template1 |    5000000 |            2142483648 |                  0.23 | 🟢 OK
```

**What to look for:**
- `pct_toward_wraparound > 90%`: **🔴 EMERGENCY** - Database shutdown imminent (hours away)
- `pct_toward_wraparound > 70%`: **🟠 CRITICAL** - Immediate action required (days away from shutdown)
- `pct_toward_wraparound > 50%`: **🟡 WARNING** - Schedule aggressive vacuum soon
- `pct_toward_wraparound < 10%`: **🟢 OK** - Autovacuum handling it normally

**Key thresholds:**
- Autovacuum aggressive mode: 200 million xids (`autovacuum_freeze_max_age`)
- Warning messages start: ~1.4 billion xids
- Emergency shutdown: 2 billion xids (`2^31`)

#### Prevention is critical

- **Monitor xid age regularly** - alert at 40-50%
- **Never disable autovacuum** - it's your protection against this
- **Don't ignore autovacuum warnings** in logs
- **Long-running transactions prevent vacuum** from advancing frozenxid
- Test your monitoring - this is a **preventable disaster**

### Monitor for Serialization Errors

If using REPEATABLE READ or SERIALIZABLE isolation levels:

```sql
SELECT
    datname,
    conflicts
FROM pg_stat_database_conflicts
WHERE conflicts > 0;
```

Check application logs for:
```
ERROR: could not serialize access due to concurrent update
ERROR: could not serialize access due to read/write dependencies
```

### Check Current Isolation Levels in Use

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    wait_event,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND pid != pg_backend_pid();
```

Note: Isolation level isn't directly visible in pg_stat_activity, but you can check your application's default:

```sql
SHOW default_transaction_isolation;
```

## Common problems

### Problem: Long-running "idle in transaction"

**Symptom**: Connection in `idle in transaction` state for hours, table bloat increasing

**Cause**:
- Application opens transaction with `BEGIN` but never commits/rolls back
- Connection pooler not properly closing transactions
- Application crash leaving transaction open

**Investigation**:
```sql
-- Find idle in transaction connections
SELECT
    pid,
    usename,
    application_name,
    now() - xact_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

**Solutions**:
1. **Immediate fix**: Terminate the connection
   ```sql
   SELECT pg_terminate_backend(12345);
   ```
2. Set `idle_in_transaction_session_timeout` to auto-kill idle transactions:
   ```sql
   ALTER DATABASE mydb SET idle_in_transaction_session_timeout = '10min';
   ```
3. Fix application code to ensure transactions are closed
4. Configure connection pooler (PgBouncer) in transaction mode

### Problem: Table bloat despite regular autovacuum

**Symptom**: Tables growing much larger than expected, queries slowing down

**Cause**: Long-running transaction preventing VACUUM from cleaning dead tuples

**Investigation**:
```sql
-- Check oldest running transaction
SELECT
    pid,
    usename,
    application_name,
    now() - xact_start AS age,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start
LIMIT 1;
```

**Solutions**:
1. Identify and terminate long transactions (see above)
2. Break long ETL/batch jobs into smaller transactions
3. Use `VACUUM FULL` as last resort (requires exclusive lock, rewrites table)
4. Consider partitioning large tables to limit bloat impact

### Problem: Transaction ID wraparound approaching

**Symptom**: Warnings in PostgreSQL logs about xid wraparound, autovacuum running aggressively

**Log messages**:
```
WARNING: database "mydb" must be vacuumed within 10000000 transactions
ERROR: database is not accepting commands to avoid wraparound data loss in database "mydb"
```

**Cause**:
- Database hasn't been vacuumed in a very long time
- Long-running transactions preventing vacuum
- Autovacuum disabled or misconfigured

**Investigation**:
```sql
-- Check databases approaching wraparound
SELECT
    datname,
    age(datfrozenxid) AS xid_age,
    2147483648 - age(datfrozenxid) AS xids_remaining
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

**Solutions**:
1. **Emergency**: Run manual VACUUM on affected databases immediately
   ```sql
   VACUUM FREEZE;  -- in each database
   ```
2. Terminate any long-running transactions
3. Ensure autovacuum is enabled and tuned appropriately
4. Monitor xid age regularly (alert at 40%)

### Problem: Serialization errors in application

**Symptom**: Application receiving `could not serialize access` errors

**Cause**: Using REPEATABLE READ or SERIALIZABLE with concurrent writes

**Investigation**:
```sql
-- Check application's isolation level
SHOW default_transaction_isolation;

-- Look for conflicting transactions
SELECT * FROM pg_stat_activity WHERE state = 'active';
```

**Solutions**:
1. Implement retry logic in application (required for SERIALIZABLE)
2. Consider using READ COMMITTED if strong consistency isn't required
3. Redesign transactions to reduce conflicts (smaller, faster transactions)
4. Add explicit locking (`SELECT ... FOR UPDATE`) where appropriate

### Problem: VACUUM taking too long or blocking operations

**Symptom**: Manual VACUUM running for hours, blocking other operations

**Cause**:
- Table is extremely bloated
- Insufficient `maintenance_work_mem`
- Using `VACUUM FULL` (requires exclusive lock)

**Investigation**:
```sql
-- Check currently running vacuum
SELECT
    pid,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE query LIKE '%VACUUM%';
```

**Solutions**:
1. Don't use `VACUUM FULL` in production - use regular `VACUUM` instead
2. Increase `maintenance_work_mem` for faster vacuum:
   ```sql
   SET maintenance_work_mem = '2GB';
   VACUUM table_name;
   ```
3. Use extensions for online table rewrites (no exclusive lock):
   - **[pg_squeeze](https://github.com/cybertec-postgresql/pg_squeeze)** - Newer extension with automatic scheduling capabilities
   - **[pg_repack](https://github.com/reorg/pg_repack)** - Well-established extension for manual table reorganization
4. Consider partitioning to make vacuum operations smaller and faster
5. Prevent bloat proactively by tuning autovacuum

## References

1. [PostgreSQL Documentation: MVCC Introduction](https://www.postgresql.org/docs/current/mvcc-intro.html)
2. [PostgreSQL Documentation: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
3. [PostgreSQL Documentation: Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
4. [PostgreSQL Documentation: Heap-Only Tuples (HOT)](https://www.postgresql.org/docs/current/storage-hot.html)
5. [The Internals of PostgreSQL: Concurrency Control](https://www.interdb.jp/pg/pgsql05.html)
6. [How to Reduce Your PostgreSQL Database Size](https://www.tigerdata.com/blog/how-to-reduce-your-postgresql-database-size)
