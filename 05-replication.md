# Replication

## What is it?

**Replication** is PostgreSQL's mechanism for maintaining one or more copies (replicas/standbys) of a database. The primary server streams WAL records to standby servers, which replay them to stay synchronized.

### Core Concepts

#### Streaming Replication (Physical Replication)

PostgreSQL's **streaming replication** replicates at the physical level (WAL records), creating exact byte-for-byte copies of the primary database.

**How it works:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PRIMARY SERVER                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Client writes data                                                 │
│         ↓                                                           │
│  ┌──────────────┐         ┌──────────────┐                          │
│  │ Transactions │────────→│   pg_wal/    │                          │
│  │              │  WAL    │  (WAL files) │                          │
│  └──────────────┘         └───────┬──────┘                          │
│                                   │                                 │
│                                   │ WAL stream                      │
└───────────────────────────────────┼─────────────────────────────────┘
                                    │
                ┌───────────────────┼───────────────────┐
                │                   │                   │
                ▼                   ▼                   ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │   STANDBY 1      │ │   STANDBY 2      │ │   STANDBY 3      │
    ├──────────────────┤ ├──────────────────┤ ├──────────────────┤
    │                  │ │                  │ │                  │
    │ WAL Receiver     │ │ WAL Receiver     │ │ WAL Receiver     │
    │       ↓          │ │       ↓          │ │       ↓          │
    │ Replay WAL       │ │ Replay WAL       │ │ Replay WAL       │
    │       ↓          │ │       ↓          │ │       ↓          │
    │ Stay in sync     │ │ Stay in sync     │ │ Stay in sync     │
    │                  │ │                  │ │                  │
    │ (Read-only)      │ │ (Read-only)      │ │ (Read-only)      │
    └──────────────────┘ └──────────────────┘ └──────────────────┘
```

**Key characteristics:**
- Standbys are **read-only** (hot standby mode)
- Exact copies of primary (same PostgreSQL version, architecture)
- Block-level replication (copies physical pages, not logical changes)
- Can have multiple standbys from single primary

**WAL transmission:**
- WAL files from `pg_wal/` are sent to standby servers using PostgreSQL's protocol
- Primary periodically recycles old WAL files (after checkpoints, when no longer needed for crash recovery)
- **Problem**: Primary may recycle WAL files before standby receives them
  - Network delays, standby downtime, or slow standby can cause standby to fall behind
  - If primary recycles WAL that standby hasn't received yet, standby cannot catch up
  - Standby would need full resynchronization (pg_basebackup) to recover
- **Solution**: Replication slots prevent this problem

#### Replication Slots

**What are replication slots?**
- Named connections that guarantee WAL retention for specific standbys
- Prevent primary from recycling WAL files that standbys still need
- Persist across restarts

**Why they matter:**
- **Without slots**: Primary may recycle WAL before standby consumes it (standby falls behind, needs full resync)
- **With slots**: Primary retains WAL until standby confirms receipt (safer, but can fill disk if standby disconnected)

**Types:**
- **Physical slots**: For streaming replication (physical standbys)
- **Logical slots**: For logical replication (subscribers)

**Critical consideration:**
- Slots prevent WAL deletion even if standby is offline
- Can cause `pg_wal/` to fill disk if standby down for extended period
- Must monitor slot lag and disk usage

#### Synchronous vs Asynchronous Replication

**Asynchronous replication (default)**
- Primary commits immediately after writing WAL locally
- WAL streamed to standby **asynchronously** (no waiting)
- **Pros**: Better performance (no network wait), standby failure doesn't affect primary
- **Cons**: Potential data loss if primary crashes before standby receives WAL

**Synchronous replication**
- Primary waits for standby to confirm WAL receipt before commit returns
- Guarantees zero data loss (standby has all committed transactions)
- **Pros**: No data loss on primary failure
- **Cons**: Higher latency (network round-trip), standby failure can block commits

**Configuration:**

```
# On primary server
synchronous_commit = on                    # Wait for sync (default: on)
synchronous_standby_names = 'standby1'     # Which standbys to wait for
```

**synchronous_commit levels:**

| Level | Wait for | Use case | Data loss risk |
|-------|----------|----------|----------------|
| `off` | Nothing | Maximum performance | High (even local crash) |
| `local` (default) | Local WAL write | Balanced | On primary failure before replication |
| `remote_write` | Standby receives WAL | Better protection | On standby OS crash |
| `on` | Standby writes WAL to disk | Zero data loss | None (on sync standby) |
| `remote_apply` | Standby applies WAL | Read-your-writes consistency | None |

> [!WARNING]
> `synchronous_commit = off` is dangerous even without replication - disables local fsync waiting. Use with extreme caution.

#### Failover and Promotion

**Failover** is the process of switching to a standby when the primary fails.

**Promotion:**
- Converting a standby to a new primary
- Standby stops replaying WAL and starts accepting writes
- **One-way operation**: promoted standby cannot rejoin as standby without rebuild

**Basic promotion:**
```bash
# On standby server (PG 12+)
pg_ctl promote -D /var/lib/postgresql/data

# Or trigger file method (older versions, PG < 12)
# Note: The trigger file path is configured via promote_trigger_file parameter
# (in recovery.conf for PG < 12, or postgresql.conf for PG 12+)
# Default location and name varies - check your configuration
touch /var/lib/postgresql/data/promote.trigger  # Example path
```

**After promotion:**
1. Update application connection strings to new primary
2. Old primary (if recoverable) must be rebuilt as standby
3. Other standbys may need reconfiguration to follow new primary

#### Rebuilding Old Primary as Standby with pg_rewind

After failover, the old primary has diverged from the new primary (different timeline). To convert it to a standby, you have two options:

**Option 1: pg_rewind (faster, recommended)**

`pg_rewind` synchronizes the old primary with the new primary by copying only the changed blocks, much faster than full rebuild.

**Requirements:**
- `wal_log_hints = on` on the old primary (or full_page_writes = on with data checksums enabled)
- Old primary must have been shut down cleanly OR can be shut down now
- Both servers accessible

**Steps:**

```bash
# 1. Stop old primary (if still running)
pg_ctl stop -D /var/lib/postgresql/data

# 2. Run pg_rewind from old primary, pointing to new primary
pg_rewind \
  --target-pgdata=/var/lib/postgresql/data \
  --source-server="host=new-primary-host port=5432 user=replication password=xxx dbname=postgres"

# 3. Configure old primary as standby
# PG 12+: Create standby.signal file
touch /var/lib/postgresql/data/standby.signal

# 4. Configure connection to new primary in postgresql.conf or postgresql.auto.conf
cat >> /var/lib/postgresql/data/postgresql.auto.conf <<EOF
primary_conninfo = 'host=new-primary-host port=5432 user=replication password=xxx'
EOF

# 5. Start old primary (now as standby)
pg_ctl start -D /var/lib/postgresql/data
```

**Option 2: pg_basebackup (slower, always works)**

If pg_rewind cannot be used (wal_log_hints was off, or timeline divergence too complex), do full rebuild:

```bash
# 1. Stop old primary
pg_ctl stop -D /var/lib/postgresql/data

# 2. Remove old data directory
rm -rf /var/lib/postgresql/data/*

# 3. Take new base backup from new primary
pg_basebackup \
  -h new-primary-host \
  -U replication \
  -D /var/lib/postgresql/data \
  -Fp -Xs -P -R

# 4. Start new standby
pg_ctl start -D /var/lib/postgresql/data
```

**When to use each:**
- **pg_rewind**: When possible - much faster (minutes vs hours for large databases)
- **pg_basebackup**: When wal_log_hints was disabled, or pg_rewind fails

#### High Availability with Patroni

The manual failover and rebuild processes described above are complex and error-prone. **Patroni** is a popular open-source HA solution that automates all of this.

**What Patroni does:**
- **Automatic failover**: Detects primary failure and promotes standby automatically
- **Automatic pg_rewind**: Rebuilds old primary as standby without manual intervention
- **Consensus-based**: Uses etcd, Consul, or ZooKeeper for distributed consensus
- **Health checks**: Continuous monitoring of primary and standbys
- **Switchover**: Planned maintenance with zero downtime
- **REST API**: Easy integration with monitoring and orchestration tools

**Key benefits:**
- Eliminates manual failover steps (no touching promote.trigger or pg_ctl promote)
- Automatic timeline management and pg_rewind execution
- Built-in fencing to prevent split-brain scenarios
- Integrates well with Kubernetes (via Zalando PostgreSQL Operator)

**Example scenario with Patroni:**

```
Without Patroni (manual):
1. Primary fails
2. DBA notices (minutes/hours)
3. DBA manually promotes standby
4. DBA updates connection strings
5. DBA runs pg_rewind on old primary
6. DBA reconfigures old primary as standby
Total time: 15-60 minutes (depending on DBA availability)

With Patroni (automatic):
1. Primary fails
2. Patroni detects failure (seconds)
3. Patroni promotes standby automatically
4. Patroni updates DCS (etcd/Consul) - clients discover new primary
5. When old primary recovers, Patroni runs pg_rewind automatically
6. Old primary rejoins as standby
Total time: 30-60 seconds (fully automated)
```

**Configuration example:**

See [patroni-config-example.yml](patroni-config-example.yml) for a complete Patroni configuration example.

**Further reading:**
- [Patroni Documentation](https://patroni.readthedocs.io/)
- [Patroni GitHub](https://github.com/zalando/patroni)

### Logical Replication (Briefly)

Unlike physical replication (entire database), **logical replication** replicates specific tables or changes based on replication identity. It uses a **publish/subscribe model** where one or more publishers send changes to one or more subscribers.

**How it works:**
- Publisher decodes WAL records into logical changes (INSERT, UPDATE, DELETE)
- Changes are sent to subscriber as SQL-like operations (not raw WAL bytes)
- Subscriber applies these changes to its own tables
- This allows cross-version replication and selective table replication

**Example flow:**
```
Publisher (Primary):
  UPDATE users SET name='Alice' WHERE id=1
         ↓
  WAL record written (physical)
         ↓
  Logical decoding plugin decodes WAL
         ↓
  Logical change: UPDATE users (id=1, name='Alice')
         ↓
  Send to subscriber
         ↓
Subscriber:
  Receives logical change
         ↓
  Executes: UPDATE users SET name='Alice' WHERE id=1
```

**Key differences from physical replication:**
- Table-level granularity (not whole database)
- **Only native way to replicate between different PostgreSQL versions** (e.g., PG 14 to PG 15)
- Allows filtering and transformation
- Higher overhead (decoding + re-executing operations)
- No automatic failover (not covered in detail here)

## Why it matters

### High Availability (HA)

- **Automatic failover**: Standby takes over if primary fails
- **Minimize downtime**: Seconds to minutes vs hours for restore from backup
- **Read scaling**: Distribute read queries across standbys (write queries still go to primary)

### Disaster Recovery (DR)

- **Geographic redundancy**: Standby in different datacenter/region
- **Protection against site failure**: Fire, power outage, network partition
- **Faster recovery**: Promote standby vs restore from backup

### Load Distribution

- **Read replicas**: Offload read-only queries to standbys
- **Reporting queries**: Run expensive analytics on standby without affecting primary
- **Backup from standby**: Take backups from standby instead of primary (reduces primary load)

### Data Protection

- **Synchronous replication**: Zero data loss guarantee
- **Multiple standbys**: Redundancy (can lose one standby and still have protection)

## How to monitor

### Check Replication Status (Primary)

```sql
-- On PRIMARY: See connected standbys and their lag
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), write_lsn) AS write_lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) AS flush_lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

**Example output:**
```
 application_name | client_addr | state     | sync_state | send_lag_bytes | write_lag_bytes | flush_lag_bytes | replay_lag_bytes | write_lag  | flush_lag  | replay_lag
------------------+-------------+-----------+------------+----------------+-----------------+-----------------+------------------+------------+------------+------------
 standby1         | 10.0.1.5    | streaming | sync       |              0 |               0 |               0 |             8192 | 00:00:00.2 | 00:00:00.2 | 00:00:00.5
 standby2         | 10.0.2.8    | streaming | async      |         524288 |          524288 |          524288 |           1048576 | 00:00:05   | 00:00:05   | 00:00:10
```

**What to look for:**
- `state = 'streaming'`: Standby is actively receiving WAL
- `state = 'catchup'`: Standby is behind and catching up
- `sync_state = 'sync'`: Synchronous standby (primary waits for this one)
- `sync_state = 'async'`: Asynchronous standby (primary doesn't wait)
- `replay_lag`: How far behind standby is in applying WAL (most important metric)
- Large lag values (>1GB or >1min): Investigate network, standby performance, or load

### Check Replication Status (Standby)

```sql
-- On STANDBY: Check if in recovery mode and replication status
SELECT
    pg_is_in_recovery() AS is_standby,
    pg_last_wal_receive_lsn() AS receive_lsn,
    pg_last_wal_replay_lsn() AS replay_lsn,
    pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS replay_lag_bytes,
    pg_last_xact_replay_timestamp() AS last_replay_time,
    NOW() - pg_last_xact_replay_timestamp() AS replication_lag_time;
```

**Example output:**
```
 is_standby | receive_lsn | replay_lsn  | replay_lag_bytes |     last_replay_time      | replication_lag_time
------------+-------------+-------------+------------------+---------------------------+---------------------
 t          | 0/6A000000  | 0/69FFF000  |             4096 | 2025-01-15 10:23:45+00    | 00:00:01.5
```

**What to look for:**
- `is_standby = true`: Server is in recovery mode (standby)
- `replay_lag_bytes`: How many bytes behind in WAL replay
- `replication_lag_time`: Time since last transaction replay
  - Growing over time = standby falling behind
  - NULL = no transactions being replayed (idle primary or severe lag)

### Monitor Replication Slots

```sql
-- On PRIMARY: Check replication slot status
SELECT
    slot_name,
    slot_type,
    active,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_wal_bytes,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

**Example output:**
```
 slot_name | slot_type | active | retained_wal_bytes | retained_wal
-----------+-----------+--------+--------------------+--------------
 standby1  | physical  | t      |           16777216 | 16 MB
 standby2  | physical  | f      |        10737418240 | 10 GB
```

**What to look for:**
- `active = false`: Slot exists but standby not connected (WAL piling up!)
- `retained_wal > 10GB`: Excessive WAL retention, check standby status
- Inactive slots with high retained WAL = disk space danger

### Check pg_wal Disk Usage

```bash
# Check pg_wal directory size
du -sh /var/lib/postgresql/data/pg_wal

# Count WAL files
ls -1 /var/lib/postgresql/data/pg_wal | wc -l
```

```sql
-- Check WAL directory size from SQL
SELECT pg_size_pretty(
    SUM(size)
) AS wal_size
FROM pg_ls_waldir();
```

### Monitor Synchronous Commit Settings

```sql
-- Check current synchronous commit configuration
SHOW synchronous_commit;
SHOW synchronous_standby_names;

-- Check which standbys are sync vs async
SELECT
    application_name,
    sync_state,
    sync_priority
FROM pg_stat_replication
ORDER BY sync_priority;
```

## Common problems

### Problem: Replication lag increasing

**Symptom**: Standby falling behind primary, replay_lag growing

**Diagnosis:**
```sql
-- On PRIMARY: Check lag for each standby
SELECT
    application_name,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) / 1024 / 1024 AS lag_mb,
    replay_lag
FROM pg_stat_replication;

-- On STANDBY: Check if CPU/IO bottleneck
SELECT * FROM pg_stat_activity WHERE backend_type = 'walreceiver' OR backend_type = 'startup';
```

**Common causes:**
1. **High write load on primary**: Standby can't keep up with WAL generation
2. **Network bandwidth**: Slow connection between primary and standby
3. **Standby resource constraints**: CPU, disk I/O, memory
4. **Long-running queries on standby**: Blocking WAL replay (hot standby conflict)
5. **Checkpoint settings**: Standby checkpointing too frequently

**Solutions:**
1. Increase standby resources (CPU, disk speed)
2. Improve network bandwidth/latency
3. Tune standby checkpoint settings:
   ```
   checkpoint_timeout = 15min
   max_wal_size = 4GB
   ```
4. Enable `hot_standby_feedback` to reduce conflicts:
   ```
   hot_standby_feedback = on  # On standby
   ```
5. Consider cascading replication (standby replicates from another standby)

### Problem: pg_wal directory filling disk

**Symptom**: Disk space alerts, pg_wal directory growing uncontrollably

**Diagnosis:**
```sql
-- Check inactive replication slots
SELECT
    slot_name,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
WHERE active = false;

-- Check WAL size
SELECT pg_size_pretty(SUM(size)) FROM pg_ls_waldir();
```

**Causes:**
- Inactive replication slot (standby disconnected but slot remains)
- Standby down for extended period
- `archive_command` failing (if archiving enabled)
- Very high write load exceeding `max_wal_size`

**Solutions:**
1. **Drop inactive slots** (if standby permanently gone):
   ```sql
   SELECT pg_drop_replication_slot('slot_name');
   ```
2. **Fix standby connection**: Investigate why standby disconnected
3. **Increase `max_wal_size` temporarily**:
   ```sql
   ALTER SYSTEM SET max_wal_size = '10GB';
   SELECT pg_reload_conf();
   ```
4. **Fix archive_command** if archiving is stuck
5. **Emergency**: Manually remove old WAL files (DANGEROUS - only if disk full and no other option):
   ```bash
   # DO NOT DO THIS unless you understand the risks
   # This will break replication - standbys will need full resync
   ```

### Problem: Synchronous standby blocking commits

**Symptom**: Transactions hanging, slow commits, timeouts

**Diagnosis:**
```sql
-- Check if sync standby is connected
SELECT
    application_name,
    sync_state,
    state
FROM pg_stat_replication
WHERE sync_state = 'sync';

-- Check synchronous settings
SHOW synchronous_commit;
SHOW synchronous_standby_names;

-- Check for blocked sessions
SELECT
    pid,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event = 'SyncRep';
```

**Causes:**
- Synchronous standby disconnected or crashed
- Network partition between primary and sync standby
- Sync standby overloaded (can't keep up)

**Solutions:**
1. **Immediate**: Disable synchronous replication (accept risk):
   ```sql
   ALTER SYSTEM SET synchronous_standby_names = '';
   SELECT pg_reload_conf();
   ```
2. **Fix standby**: Restart standby, investigate why it stopped responding
3. **Add more sync standbys**: Use `ANY` or `FIRST` syntax for redundancy:
   ```
   synchronous_standby_names = 'FIRST 1 (standby1, standby2)'  # Wait for any one
   ```
4. **Lower synchronous_commit** temporarily:
   ```sql
   SET synchronous_commit = local;  # Per-session, accept some data loss risk
   ```

### Problem: Hot standby conflicts (queries canceled on standby)

**Symptom**: Read queries on standby canceled with "conflict with recovery" errors

**Example error:**
```
ERROR: canceling statement due to conflict with recovery
DETAIL: User query might have needed to see row versions that must be removed.
```

**Cause:**
- Primary vacuums or updates rows that standby queries are reading
- Standby WAL replay needs to remove row versions that queries need

**Diagnosis:**
```sql
-- On STANDBY: Check conflict statistics
SELECT
    datname,
    confl_tablespace,
    confl_lock,
    confl_snapshot,
    confl_bufferpin,
    confl_deadlock
FROM pg_stat_database_conflicts;
```

**Solutions:**
1. **Enable hot_standby_feedback** (primary won't vacuum rows standby needs):
   ```
   hot_standby_feedback = on  # On standby postgresql.conf
   ```
   - Tradeoff: Can cause bloat on primary if long queries on standby
2. **Increase max_standby_streaming_delay** (give queries more time):
   ```
   max_standby_streaming_delay = 300s  # On standby, default: 30s
   ```
3. **Run long queries on dedicated standby**: Keep one standby for reporting queries
4. **Use logical replication**: For reporting workloads that need no conflicts

### Problem: Standby won't start after promotion failure

**Symptom**: Tried to promote standby, something went wrong, now it won't start as standby or primary

**Diagnosis:**
```bash
# Check PostgreSQL logs
tail -100 /var/log/postgresql/postgresql-*.log

# Check for standby.signal or recovery.signal file
ls -la /var/lib/postgresql/data/*.signal
```

**Causes:**
- Partial promotion (standby.signal removed but timeline not advanced)
- WAL divergence (standby has different timeline than primary)
- Corruption during promotion

**Solutions:**
1. **If promoting**: Complete the promotion:
   ```bash
   pg_ctl promote -D /var/lib/postgresql/data
   ```
2. **If reverting to standby**: Rebuild from primary (pg_basebackup)
3. **Check timeline**: Ensure standby timeline matches primary or is direct successor

## References

1. [PostgreSQL Documentation: High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)
2. [PostgreSQL Documentation: Replication](https://www.postgresql.org/docs/current/runtime-config-replication.html)
3. [PostgreSQL Documentation: pg_stat_replication](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-REPLICATION-VIEW)
4. [PostgreSQL Documentation: Replication Slots](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION-SLOTS)
5. [PostgreSQL Documentation: Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html)
