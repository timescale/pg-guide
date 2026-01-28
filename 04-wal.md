# WAL (Write-Ahead Log)

## What is it?

**WAL (Write-Ahead Log)** is PostgreSQL's mechanism for ensuring data durability and enabling crash recovery. The fundamental principle: **write changes to a sequential log before modifying data files**.

### Core Concepts

#### Write-Ahead Logging Principle

**Rule: Log the change BEFORE applying it to data files on disk**

```
When you modify data (even before COMMIT):

┌──────────────────────┐
│  UPDATE users        │
│  SET balance = 100   │
│  WHERE id = 1;       │
└──────────┬───────────┘
           │
           ├─────────────────────┐
           │                     │
           ▼                     ▼
  ┌─────────────────┐   ┌─────────────────┐
  │   pg_wal/       │   │ Shared Buffers  │
  │   (disk)        │   │   (memory)      │
  ├─────────────────┤   ├─────────────────┤
  │ WAL record:     │   │ Page modified:  │
  │ "UPDATE bal=100"│   │ balance = 100   │
  │                 │   │                 │
  │ ✓ Written       │   │ ✓ Updated       │
  │   immediately   │   │   immediately   │
  └─────────────────┘   └─────────────────┘
           │                     │
           │                     │
           └──────────┬──────────┘
                      │
                      │ COMMIT happens:
                      │ → Flush WAL to disk (fsync)
                      │ → Transaction durable ✓
                      │
                      ▼
           ┌─────────────────────┐
           │ Data files on disk  │
           │ (base/...)          │
           ├─────────────────────┤
           │ balance = 50 (old)  │
           │                     │
           │ ⏳ NOT updated yet! │
           │                     │
           │ Will be written     │
           │ later by:           │
           │ • Checkpoint        │
           │ • Background writer │
           └─────────────────────┘

Key points:
• pg_wal and shared_buffers updated TOGETHER during the transaction
• Data file update is LAZY (happens much later)
• COMMIT only waits for WAL flush (fast!)
• If crash: WAL replay reconstructs changes to data files
```

#### WAL Segments

- WAL is stored in **16MB files** called segments (default size)

> [!NOTE]
> Segment size can be changed using `initdb --wal-segsize` option (PostgreSQL 11+). This must be set during database initialization - cannot be changed later without re-initializing the cluster. Most installations use the default 16MB. See [initdb documentation](https://www.postgresql.org/docs/current/app-initdb.html#APP-INITDB-OPTION-WAL-SEGSIZE).
- Located in `pg_wal/` directory (formerly `pg_xlog/` in PG < 10)
- Files named with 24-character hexadecimal names
- PostgreSQL recycles old WAL files instead of deleting them
- Number of WAL files depends on activity and checkpoint settings

#### WAL File Name Structure

A typical WAL file name, such as `0000000100001234000000AB`, is a 24-character hexadecimal string composed of three parts:

```
0000000100001234000000AB
│       │       │       
│       │       └──────── Segment number (low 32 bits)
│       └──────────────── Logical file number (high 32 bits)
└──────────────────────── Timeline ID
│       
└──────────────────────── 24 hex chars total

Breaking down:  00000001   0000001234  000000AB
               └───┬────┘  └───┬────┘ └───┬────┘
               Timeline    Log File    Segment
```

**Components:**

1. **Timeline ID** (first 8 hex digits: `TTTTTTTT`)
   - Represents the timeline ID
   - Starts at `00000001` for initial database
   - Increments during point-in-time recovery (PITR) to manage different database history branches
   - Example: After PITR, timeline becomes `00000002`

2. **Logical WAL file number** (next 8 hex digits: `XXXXXXXX`)
   - High 32 bits of the 64-bit log sequence number (LSN)
   - Indicates sequential file number within the timeline
   - Increments as WAL grows

3. **Segment file number** (final 8 hex digits: `YYYYYYYY`)
   - Low 32 bits of the segment number
   - Physical segment within the logical file
   - With default 16MB segments, this cycles from `00000000` to `000000FF` (256 segments)
   - Then logical file number increments and segment resets to `00000000`

**Examples:**

```
000000010000000000000001  →  Timeline 1, first WAL file, segment 1
000000010000000000000002  →  Timeline 1, first WAL file, segment 2
00000001000000000000000FF →  Timeline 1, first WAL file, segment 255
000000010000000100000000  →  Timeline 1, second WAL file, segment 0
000000020000000000000001  →  Timeline 2 (after PITR), first file, segment 1
```

**Why this matters:**
- Understanding file names helps with WAL archiving and recovery troubleshooting
- Sequential names ensure ordered replay during recovery
- Timeline changes are visible in file names (helps detect PITR events)

#### WAL Records

- Each transaction generates WAL records describing changes
- **Important**: WAL records are written for ALL changes, including uncommitted transactions
  - This means even if a transaction rolls back, its changes were logged to WAL
  - Rollback itself generates WAL records (to undo the changes)
  - This is necessary for crash recovery - PostgreSQL needs to know about in-progress transactions
- Records are sequential and ordered
- Include enough information to redo (replay) the operation
- Very compact: only describes the change, not full page images (usually)

#### Durability Guarantees

**fsync** (default: on)
- Forces WAL writes to physical disk before commit returns
- Ensures true durability (survives power loss)
- **Never disable in production** - risks data loss

**wal_sync_method**
- Controls the method used to force WAL updates to disk
- Platform-dependent; PostgreSQL chooses the best default for your OS
- Common options:
  - `fsync` (default on most systems): Call fsync() after write
  - `fdatasync` (Linux): Like fsync but doesn't update metadata timestamps (slightly faster)
  - `open_datasync`: Open file with O_DSYNC flag (write is synchronous)
  - `open_sync`: Open file with O_SYNC flag
  - `fsync_writethrough` (Windows): Forces write-through of disk cache
- **Note**: Only change if you have specific performance issues and understand the trade-offs
- Wrong setting can cause data corruption on crash
- Use `pg_test_fsync` utility to test which method is fastest on your hardware

## Why it matters

### Crash Recovery

WAL enables PostgreSQL to recover from crashes:

1. **During normal operation**: Changes written to WAL, then to data files
2. **Crash occurs**: Data files may be incomplete/inconsistent
3. **On restart**: PostgreSQL replays WAL from last checkpoint
4. **Result**: Database restored to consistent state

Without WAL, crash would corrupt database permanently.

### Performance Benefits

**Sequential writes are fast**
- WAL writes are sequential (append-only)
- Much faster than random writes to data files
- Enables batching: many transactions commit with single WAL flush

**Delayed data file writes**
- Data files can be written lazily (by background writer/checkpointer)
- Allows multiple changes to same page to be combined
- Reduces total I/O

### Replication Foundation

- **Streaming replication**: Standby servers replay WAL from primary
- **Logical replication**: Decode WAL to extract logical changes
- **PITR (Point-in-Time Recovery)**: Archive WAL for restore to any point

## How it works

### WAL Write Flow

```
┌────────────────────────────────┬──────────────────────────────────────┐
│      TRANSACTION               │           STORAGE                    │
├────────────────────────────────┼──────────────────────────────────────┤
│                                │                                      │
│ BEGIN;                         │                                      │
│                                │                                      │
│ UPDATE users SET name='Alice'  │                                      │
│                                │                                      │
│ Step 1: Generate WAL record ───┼─>┌─────────────────────────────┐     │
│                                │  │ WAL Buffers (shared memory) │     │
│                                │  │ wal_buffers (default: auto) │     │
│                                │  │                             │     │
│                                │  │ "UPDATE users xid=100..."   │     │
│                                │  │ [Buffered in memory]        │     │
│                                │  └─────────────────────────────┘     │
│                                │                                      │
│ Step 2: Modify page in ────────┼─>┌─────────────────────────────┐     │
│         shared_buffers         │  │ Shared Buffers (memory)     │     │
│                                │  │                             │     │
│                                │  │ users page (dirty)          │     │
│                                │  └─────────────────────────────┘     │
│                                │                                      │
│ COMMIT;                        │                                      │
│                                │          ↓                           │
│ Step 3: Flush WAL buffers ─────┼─>┌─────────────────────────────┐     │
│         to disk (fsync)        │  │ pg_wal/0000...0001 (disk)   │     │
│                                │  │                             │     │
│                                │  │ [WAL record persisted] ✓    │     │
│ ← COMMIT returns SUCCESS       │  └─────────────────────────────┘     │
│   (transaction is durable)     │                                      │
│                                │                                      │
│                                │    [Much later: checkpoint or        │
│                                │     background writer]               │
│                                │          ↓                           │
│ Step 4: Lazy write to disk ────┼─>┌─────────────────────────────┐     │
│         (async)                │  │ Data files (disk)           │     │
│                                │  │ base/12345/67890            │     │
│                                │  │                             │     │
│                                │  │ [Dirty page flushed]        │     │
│                                │  └─────────────────────────────┘     │
└────────────────────────────────┴──────────────────────────────────────┘

Key insights:
• WAL records first go to WAL buffers (shared memory, size: wal_buffers)
• WAL writer periodically flushes buffers to disk (every wal_writer_delay)
• COMMIT forces immediate flush of WAL buffers to disk (fsync)
• COMMIT returns only after WAL is safely on disk (durable)
• Data file update happens much later (async, via checkpoint/bgwriter)
• If crash occurs, WAL replay reconstructs changes to data files
```

#### Understanding Dirty Pages

**What is a dirty page?**
- A "dirty page" is a page in shared_buffers that has been modified in memory but not yet written to the data files on disk
- When you UPDATE/INSERT/DELETE, the page is modified in shared_buffers (becomes "dirty")
- The page stays dirty until a checkpoint or background writer flushes it to disk

**Why this matters:**
- WAL protects us: even if dirty pages are lost in a crash, we can replay WAL to recreate them
- Multiple changes to the same page can accumulate in memory before being written to disk (more efficient)
- Too many dirty pages = longer recovery time (more WAL to replay)
- Checkpoints exist to periodically flush dirty pages and limit recovery time

**Example:**
```
UPDATE users SET balance = 100 WHERE id = 1;
                ↓
shared_buffers: page is DIRTY (modified in memory)
data files:     page is OLD (not updated yet)
pg_wal:         change is LOGGED (safe on disk)
                ↓
[Checkpoint happens]
                ↓
shared_buffers: page is CLEAN (matches disk)
data files:     page is UPDATED ✓
```

### Checkpoints

**What is a checkpoint?**

A checkpoint is a point where:
1. All dirty pages in shared_buffers are flushed to disk
2. A checkpoint record is written to WAL
3. Recovery can start from this point (don't need to replay earlier WAL)

**Why checkpoints matter:**

- **Shorter recovery**: Only replay WAL since last checkpoint
- **WAL recycling**: Can recycle WAL segments before last checkpoint
- **I/O spike**: Flushing all dirty pages causes I/O burst

**Checkpoint frequency controlled by:**

```
checkpoint_timeout = 5min          # Time-based checkpoint (default)
max_wal_size = 1GB                 # Size-based checkpoint (default)
```

Checkpoint happens when **either** condition is met.

**Checkpoint spreading:**

```
checkpoint_completion_target = 0.9  # Spread checkpoint I/O over 90% of interval
```

Spreads checkpoint writes over time to reduce I/O spikes.

**Example calculation:**
- `checkpoint_timeout = 5min` (300 seconds)
- `checkpoint_completion_target = 0.9`
- **Checkpoint duration**: 300s × 0.9 = **270 seconds (4min 30s)**
- The checkpoint will spread its I/O over 4min 30s instead of writing everything at once
- This leaves 30 seconds buffer before the next checkpoint starts (avoids overlapping checkpoints)

**Warning: Checkpoints occurring too frequently**

If you see this warning in PostgreSQL logs:

```
LOG:  checkpoints are occurring too frequently (9 seconds apart)
HINT:  Consider increasing the configuration parameter "max_wal_size".
LOG:  checkpoints are occurring too frequently (2 seconds apart)
HINT:  Consider increasing the configuration parameter "max_wal_size".
```

**What it means:**
- Checkpoints are happening because `max_wal_size` is being reached (not `checkpoint_timeout`)
- WAL generation is too fast for current settings
- Frequent checkpoints cause I/O spikes and performance degradation
- You're not getting the benefit of `checkpoint_completion_target` spreading

**Solutions:**
1. **Increase `max_wal_size`**: Allow more WAL to accumulate between checkpoints
   ```
   max_wal_size = 4GB  # Increase from default 1GB
   ```
2. **Increase `checkpoint_timeout`**: Space checkpoints further apart in time
   ```
   checkpoint_timeout = 15min  # Increase from default 5min
   ```
3. Monitor disk space in `pg_wal/` directory to ensure you have enough space for larger WAL size

**Further reading:** [Tuning Your Postgres Database for High Write Loads](https://www.crunchydata.com/blog/tuning-your-postgres-database-for-high-write-loads) - comprehensive guide on checkpoint tuning and write-heavy workload optimization

### WAL Configuration Parameters

#### wal_level (restart required)

Controls how much information is written to WAL:

- **minimal**: Only enough for crash recovery (no archiving, no replication)
- **replica**: Enables WAL archiving and streaming replication (default)
- **logical**: Enables logical replication decoding

**Use `replica` for production** (enables backups and replication).

#### wal_buffers

- Amount of shared memory for WAL buffering
- Default: -1 (auto-sized to 1/32 of shared_buffers, max 16MB)
- Rarely needs tuning (auto-sizing works well)

#### wal_writer_delay

- How often WAL writer flushes WAL to disk (default: 200ms)
- For very high transaction rates, may reduce to 10ms
- Most workloads: default is fine

#### min_wal_size / max_wal_size

- **min_wal_size** (default: 80MB): Always keep this much WAL
- **max_wal_size** (default: 1GB): Trigger checkpoint when WAL exceeds this
- **Important**: max_wal_size is a **soft limit** - WAL can grow beyond it during heavy load

## How to monitor

### Check Current WAL Location

```sql
SELECT pg_current_wal_lsn();
```

**Example output:**
```
 pg_current_wal_lsn
--------------------
 0/1A2B3C4D
```

WAL LSN (Log Sequence Number) is a pointer to a position in the WAL stream.

### Monitor WAL Generation Rate

```sql
SELECT
    pg_current_wal_lsn(),
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 1024 / 1024 AS wal_mb_generated;
```

**To measure rate over time:**
```sql
-- Take snapshot 1
SELECT pg_current_wal_lsn() AS lsn1, now() AS time1 \gset

-- Wait 60 seconds (or run this later)
\! sleep 60

-- Take snapshot 2 and calculate rate
SELECT
    pg_current_wal_lsn() AS lsn2,
    now() AS time2,
    pg_wal_lsn_diff(pg_current_wal_lsn(), :'lsn1') / 1024 / 1024 AS wal_mb_generated,
    pg_wal_lsn_diff(pg_current_wal_lsn(), :'lsn1') / 1024 / 1024 /
        EXTRACT(EPOCH FROM (now() - :'time1'::timestamp)) AS wal_mb_per_sec;
```

### Check WAL Files on Disk

```sql
SELECT
    count(*) AS wal_files,
    sum((pg_stat_file('pg_wal/' || name)).size) / 1024 / 1024 AS total_wal_mb
FROM pg_ls_waldir();
```

**Example output:**
```
 wal_files | total_wal_mb
-----------+--------------
        64 |         1024
```

**What to look for:**
- WAL file count growing rapidly: Heavy write load or checkpoint tuning needed
- WAL directory filling disk: Check `max_wal_size` and disk space

### Monitor Checkpoints

```sql
SELECT
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend
FROM pg_stat_bgwriter;
```

**Example output:**
```
 checkpoints_timed | checkpoints_req | checkpoint_write_time | checkpoint_sync_time | buffers_checkpoint | buffers_clean | buffers_backend
-------------------+-----------------+-----------------------+----------------------+--------------------+---------------+-----------------
              1234 |              56 |             45678.123 |              234.567 |           12345678 |       5678901 |         1234567
```

**What to look for:**
- **checkpoints_req > checkpoints_timed**: Too many size-based checkpoints - increase `max_wal_size`
- **High checkpoint_write_time**: Checkpoints taking too long - consider increasing `checkpoint_completion_target`
- **buffers_backend high**: Backends writing pages themselves - increase shared_buffers or tune checkpointer

**Calculate checkpoint frequency:**
```sql
SELECT
    checkpoints_timed,
    checkpoints_req,
    ROUND(100.0 * checkpoints_req / NULLIF(checkpoints_timed + checkpoints_req, 0), 2) AS pct_req_checkpoints
FROM pg_stat_bgwriter;
```

**Ideal**: `pct_req_checkpoints < 10%` (most checkpoints should be time-based, not size-based)

### Monitor WAL Archiving (if enabled)

```sql
SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_archived_time,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

**Example output:**
```
 archived_count | failed_count |     last_archived_wal      |       last_archived_time       | last_failed_wal | last_failed_time
----------------+--------------+----------------------------+--------------------------------+-----------------+------------------
         123456 |            0 | 000000010000001A0000002B   | 2026-01-22 15:30:45.123+00     |                 |
```

**What to look for:**
- **failed_count > 0**: Archive command failing - check logs and archive destination
- **last_archived_time old**: Archiving falling behind - check archive destination capacity

### Check WAL Configuration

```sql
SELECT
    name,
    setting,
    unit,
    context
FROM pg_settings
WHERE name IN (
    'wal_level',
    'fsync',
    'synchronous_commit',
    'wal_buffers',
    'checkpoint_timeout',
    'max_wal_size',
    'min_wal_size',
    'checkpoint_completion_target'
)
ORDER BY name;
```

## Common problems

### Problem: WAL directory filling disk

**Symptom**: Disk space running out in `pg_wal/` directory, database performance degraded or stopped

**Causes**:
- Replication slot preventing WAL recycling (standby lagging or disconnected)
- WAL archiving failing or too slow
- Very high write load exceeding `max_wal_size`

**Investigation**:
```sql
-- Check WAL disk usage
SELECT count(*) AS wal_files,
       sum((pg_stat_file('pg_wal/' || name)).size) / 1024 / 1024 AS total_mb
FROM pg_ls_waldir();

-- Check replication slots (can prevent WAL removal)
SELECT
    slot_name,
    slot_type,
    active,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) / 1024 / 1024 AS mb_behind
FROM pg_replication_slots;

-- Check archiving status
SELECT * FROM pg_stat_archiver;
```

**Solutions**:
1. **Immediate**: Free up disk space temporarily
2. Check for inactive/stuck replication slots: `SELECT pg_drop_replication_slot('slot_name');`
3. Fix archive command if failing
4. Increase `max_wal_size` if write load is high
5. Add more disk space to WAL partition

### Problem: Too many requested checkpoints

**Symptom**: `checkpoints_req` much higher than `checkpoints_timed`, log messages about checkpoints

**Log example**:
```
LOG: checkpoints are occurring too frequently (20 seconds apart)
HINT: Consider increasing the configuration parameter "max_wal_size".
```

**Cause**: WAL generation rate exceeds `max_wal_size`, forcing frequent checkpoints

**Investigation**:
```sql
SELECT
    checkpoints_timed,
    checkpoints_req,
    ROUND(100.0 * checkpoints_req / (checkpoints_timed + checkpoints_req), 2) AS pct_req
FROM pg_stat_bgwriter;
```

**Solutions**:
1. Increase `max_wal_size`:
   ```sql
   ALTER SYSTEM SET max_wal_size = '2GB';
   SELECT pg_reload_conf();
   ```
2. Consider increasing `checkpoint_timeout` (but be aware: longer recovery time)
3. Check if write load is expected or if there's a runaway query

### Problem: Checkpoint I/O spikes causing performance issues

**Symptom**: Regular performance degradation every few minutes, coinciding with checkpoints

**Investigation**:
```sql
-- Check checkpoint timing and I/O
SELECT
    checkpoints_timed + checkpoints_req AS total_checkpoints,
    checkpoint_write_time / 1000.0 / 60 AS checkpoint_write_minutes,
    checkpoint_sync_time / 1000.0 AS checkpoint_sync_seconds
FROM pg_stat_bgwriter;
```

**Solutions**:
1. Increase `checkpoint_completion_target` (spread checkpoint I/O):
   ```sql
   ALTER SYSTEM SET checkpoint_completion_target = 0.9;
   ```
2. Increase `checkpoint_timeout` (less frequent checkpoints):
   ```sql
   ALTER SYSTEM SET checkpoint_timeout = '15min';
   ```
3. Increase `max_wal_size` to reduce checkpoint frequency
4. Consider faster storage (SSDs) or tune OS I/O scheduler

### Problem: WAL archiving falling behind

**Symptom**: Archive directory not receiving files, `pg_stat_archiver` shows old `last_archived_time`

**Investigation**:
```sql
SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_archived_time,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

**Common causes**:
- Archive destination disk full
- Archive command permissions issue
- Network issues to archive destination
- Archive command bug/misconfiguration

**Check PostgreSQL logs for archive command errors:**
```bash
grep "archive command failed" /var/log/postgresql/postgresql-*.log
```

**Solutions**:
1. Fix archive destination (free space, permissions, network)
2. Test archive command manually:
   ```bash
   archive_command='cp %p /archive/%f'
   # Test it:
   cp /var/lib/postgresql/data/pg_wal/000000010000000000000001 /archive/
   ```
3. Temporarily disable archiving if recovery is not critical (not recommended):
   ```sql
   ALTER SYSTEM SET archive_mode = off;  -- Requires restart!
   ```

### Problem: Database crash recovery taking too long

**Symptom**: After crash, PostgreSQL takes many minutes/hours to start

**Cause**: Long checkpoint interval means lots of WAL to replay

**Investigation**: Check PostgreSQL logs during recovery
```
LOG: database system was interrupted; last known up at 2026-01-22 10:00:00 UTC
LOG: redo starts at 0/1A000028
LOG: redo done at 0/5F2A3B4C system usage: CPU: user: 45.23 s, system: 12.34 s, elapsed: 1234.56 s
```

**Solutions**:
1. **Preventive**: Tune checkpoint settings for balance:
   ```sql
   -- More frequent checkpoints = shorter recovery
   ALTER SYSTEM SET checkpoint_timeout = '5min';
   ALTER SYSTEM SET max_wal_size = '1GB';
   ```
2. Use faster storage (NVMe SSDs) for WAL and data directories
3. Consider if crash is due to hardware/OS issues that need fixing

### Problem: Replication lag due to WAL generation rate

**Symptom**: Standby servers falling behind, replication lag increasing

**Investigation**:
```sql
-- On primary: measure WAL generation rate
SELECT pg_current_wal_lsn();
-- Run again after 60 seconds and calculate difference

-- Check standby lag
SELECT
    client_addr,
    state,
    pg_wal_lsn_diff(sent_lsn, flush_lsn) / 1024 / 1024 AS lag_mb
FROM pg_stat_replication;
```

**Causes**:
- Write load too high for standby to keep up
- Network bandwidth limitation
- Standby hardware slower than primary

**Solutions**:
1. Upgrade standby hardware or network
2. Reduce write load on primary if possible
3. Use `wal_compression = on` to reduce WAL size (PG 9.5+)
4. Consider multiple standbys for load distribution

## References

1. [PostgreSQL Documentation: WAL Configuration](https://www.postgresql.org/docs/current/wal-configuration.html)
2. [PostgreSQL Documentation: WAL Internals](https://www.postgresql.org/docs/current/wal-internals.html)
3. [PostgreSQL Documentation: Reliability and the Write-Ahead Log](https://www.postgresql.org/docs/current/wal-intro.html)
4. [PostgreSQL Documentation: pg_test_fsync](https://www.postgresql.org/docs/current/pgtestfsync.html) - Tool to test wal_sync_method performance
5. [PostgreSQL Documentation: wal_sync_method parameter](https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-WAL-SYNC-METHOD)
6. [The Internals of PostgreSQL: WAL](https://www.interdb.jp/pg/pgsql09.html)
