# Tablespaces and Storage

## What is it?

PostgreSQL's storage layer organizes data files on disk in a structured way. Understanding this physical layout helps with capacity planning, performance tuning, and troubleshooting storage issues.

**Important terminology:**

In PostgreSQL documentation, **"cluster"** refers to a single PostgreSQL instance (one postmaster process) managing one PGDATA directory that can contain multiple databases. This is different from high-availability "cluster" terminology:

- **PostgreSQL cluster** = One instance, one PGDATA directory, multiple databases
  - Example: One PostgreSQL server with databases `app_db`, `reporting_db`, `test_db`
- **HA cluster** (Patroni, Kubernetes, etc.) = Multiple PostgreSQL instances (primary + standbys)
  - Example: 3-node Patroni cluster with 1 primary + 2 standbys

Throughout this chapter, "cluster" means **PostgreSQL cluster** (single instance), not HA cluster.

### Core Concepts

#### PGDATA Directory Structure

The **PGDATA** directory (often `/var/lib/postgresql/data`) contains all database files and configuration.

**Key directories:**

```
PGDATA/
├── base/              # Database files (one subdirectory per database)
├── global/            # Cluster-wide tables (pg_database, pg_authid, etc)
├── pg_wal/            # Write-Ahead Log files
├── pg_xact/           # Transaction commit status
├── pg_multixact/      # Multixact status (for row-level locking)
├── pg_notify/         # LISTEN/NOTIFY queue
├── pg_snapshots/      # Exported snapshots
├── pg_stat/           # Permanent statistics files
├── pg_stat_tmp/       # Temporary statistics files
├── pg_subtrans/       # Subtransaction status
├── pg_tblspc/         # Symbolic links to tablespaces
├── pg_twophase/       # Prepared transaction state
├── postgresql.conf    # Main configuration file
├── pg_hba.conf        # Client authentication configuration
└── postmaster.pid     # PID file (present when server running)
```

**Note:** Configuration files (`postgresql.conf`, `pg_hba.conf`) are typically located in PGDATA but can be moved to external locations if needed. See [Chapter 10: Configuration and Tuning](10-configuration-tuning.md) for details on configuration file management, including external config files and include directives.

#### Database Files (base/)

Each database has a subdirectory under `base/` named by its OID:

```bash
$ ls -la /var/lib/postgresql/data/base/
drwx------ 2 postgres postgres 4096 Jan 15 10:00 1      # template1
drwx------ 2 postgres postgres 4096 Jan 15 10:00 13395  # template0
drwx------ 2 postgres postgres 4096 Jan 15 10:01 16384  # postgres
drwx------ 2 postgres postgres 8192 Jan 15 10:15 16385  # myapp_db
```

**Inside each database directory:**
- Each table, index, and other relation has one or more files
- Files named by relation OID (pg_class.relfilenode)
- Files split into 1GB segments: `16384`, `16384.1`, `16384.2`, etc.

**Example:**
```bash
$ ls -lh base/16385/
-rw------- 1 postgres postgres 1.0G Jan 15 10:20 16400      # Table segment 0
-rw------- 1 postgres postgres 512M Jan 15 10:20 16400.1    # Table segment 1
-rw------- 1 postgres postgres 8.0K Jan 15 10:15 16400_fsm  # Free Space Map
-rw------- 1 postgres postgres 24K  Jan 15 10:15 16400_vm   # Visibility Map
```

**File types:**

- **Main file** (`16400`): Contains the actual table or index data pages[^1]

- **Free Space Map** (`_fsm`): Tracks available free space in each page[^2]
  - Used by INSERT to find pages with enough space for new rows
  - Updated by VACUUM when dead tuples are removed
  - Each bit represents approximate free space in a page
  - Without FSM, INSERTs would need to scan entire table to find free space
  - Small file size (typically few KB even for large tables)
  - **Note**: FSM tracks space that exists - how much space to reserve is controlled by table's `fillfactor` setting (see Fill Factor section below)

- **Visibility Map** (`_vm`): Tracks which pages contain only visible tuples (no dead tuples)[^3]
  - Each bit represents one page (1 bit = "all tuples visible to all transactions")
  - **Critical for VACUUM optimization**: VACUUM can skip pages marked as all-visible[^4]
  - **Index-only scans**: If VM shows page is all-visible, no need to check heap
  - Updated during VACUUM when page becomes all-visible
  - Also tracks pages that are frozen (all tuples have frozen xids)
  - Small file size (1 bit per page = 1 byte per 8 pages)

#### Tablespaces

**Tablespaces** allow storing database objects in different filesystem locations.

**Default tablespaces:**
- `pg_default`: Default tablespace (stored in `PGDATA/base/`)
- `pg_global`: Cluster-wide tables (stored in `PGDATA/global/`)

#### Cluster-Wide Tables (Shared Catalogs)

The `pg_global` tablespace contains **shared system catalogs** - tables that are shared across all databases in the PostgreSQL cluster (not specific to any single database).

**Key shared catalogs:**

- **pg_database**: List of all databases in the cluster
  ```sql
  SELECT datname, datdba, encoding FROM pg_database;
  ```

- **pg_authid**: Roles and users (accessed via pg_user, pg_shadow views)
  ```sql
  SELECT rolname, rolsuper, rolcanlogin FROM pg_authid;
  ```

- **pg_auth_members**: Role membership (which roles belong to which roles)
  ```sql
  SELECT * FROM pg_auth_members;
  ```

- **pg_tablespace**: Tablespace definitions
  ```sql
  SELECT spcname, pg_tablespace_location(oid) FROM pg_tablespace;
  ```

- **pg_db_role_setting**: Database and role-specific configuration settings
- **pg_shdepend**: Shared object dependencies
- **pg_shdescription**: Comments on shared objects
- **pg_shseclabel**: Security labels on shared objects

**Why stored in pg_global?**

These tables exist at the **cluster level** rather than database level:
- A role created with `CREATE ROLE` exists across all databases (can connect to any database if granted permission)
- Databases themselves are listed here (you can't store the database list inside a specific database)
- Tablespace definitions are cluster-wide (can be used by any database)

**Example:**
```sql
-- Create a role (stored in pg_global/pg_authid)
CREATE ROLE app_user WITH LOGIN PASSWORD 'secret';

-- This role can now connect to ANY database in the cluster
-- (assuming permissions are granted)
```

**Viewing shared catalog locations:**
```bash
$ ls -lh /var/lib/postgresql/data/global/
-rw------- 1 postgres postgres  8.0K Jan 15 10:00 1260  # pg_authid
-rw------- 1 postgres postgres  8.0K Jan 15 10:00 1262  # pg_database
-rw------- 1 postgres postgres  8.0K Jan 15 10:00 1213  # pg_tablespace
```

**Custom tablespaces:**
- Created with `CREATE TABLESPACE`
- Symbolic link in `PGDATA/pg_tblspc/` points to actual location
- Useful for separating hot data to faster storage (SSD) or large archives to slower storage

**Example:**
```sql
-- Create tablespace on SSD mount
CREATE TABLESPACE fast_storage LOCATION '/mnt/ssd/pg_data';

-- Create table in custom tablespace
CREATE TABLE hot_data (
    id SERIAL PRIMARY KEY,
    data TEXT
) TABLESPACE fast_storage;

-- Move existing table to different tablespace
ALTER TABLE large_archive SET TABLESPACE slow_storage;
```

**Physical layout:**
```bash
$ ls -la /var/lib/postgresql/data/pg_tblspc/
lrwxrwxrwx 1 postgres postgres 20 Jan 15 10:30 16387 -> /mnt/ssd/pg_data
lrwxrwxrwx 1 postgres postgres 22 Jan 15 10:35 16388 -> /mnt/hdd/pg_archive
```

#### TOAST (The Oversized-Attribute Storage Technique)

**TOAST** is PostgreSQL's mechanism for storing large column values that don't fit in a single page (8KB default).

**How it works:**

```
Normal row (fits in 8KB page):
┌─────────────────────────────────────┐
│ Page (8KB)                          │
├─────────────────────────────────────┤
│ Row: id=1, name='Alice', data='...' │
│ Row: id=2, name='Bob', data='...'   │
│ Row: id=3, name='Carol', data='...' │
└─────────────────────────────────────┘

Row with large column (TOAST):
┌─────────────────────────────────────┐
│ Main Table Page                     │
├─────────────────────────────────────┤
│ Row: id=1, name='Alice',            │
│      large_data=[TOAST POINTER]  ───┼──┐
└─────────────────────────────────────┘  │
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │ TOAST Table          │
                              ├──────────────────────┤
                              │ Chunk 1: [2KB data]  │
                              │ Chunk 2: [2KB data]  │
                              │ Chunk 3: [2KB data]  │
                              │ Chunk 4: [2KB data]  │
                              └──────────────────────┘
```

**TOAST strategies:**

| Strategy | Description | When used |
|----------|-------------|-----------|
| `PLAIN` | No compression or out-of-line storage | Small fixed-length types (int, date) |
| `EXTENDED` | Compress and out-of-line if needed (default) | Most variable-length types (text, bytea) |
| `EXTERNAL` | Out-of-line but no compression | Already compressed data (images, compressed text) |
| `MAIN` | Compress but avoid out-of-line | Prefer keeping in main table for performance |

**Set TOAST strategy:**
```sql
-- For already compressed data (skip compression overhead)
ALTER TABLE images ALTER COLUMN photo_data SET STORAGE EXTERNAL;

-- For frequently accessed small text (avoid out-of-line)
ALTER TABLE users ALTER COLUMN email SET STORAGE MAIN;
```

**TOAST tables:**
- Automatically created per-table as `pg_toast.pg_toast_<oid>`
- Not visible in normal queries
- Can bloat like regular tables (needs VACUUM)

#### Fill Factor

**fillfactor** controls how full pages are packed during INSERT/UPDATE.

**Default: 100% (completely full pages)**

```
fillfactor = 100 (default for most tables):
┌──────────────────────────┐
│ Page (8KB)               │
├──────────────────────────┤
│ Row 1                    │
│ Row 2                    │
│ Row 3                    │
│ Row 4                    │
│ ... (packed full)        │
│ No free space            │
└──────────────────────────┘
UPDATE needs new page → creates bloat

fillfactor = 70 (30% free space reserved):
┌──────────────────────────┐
│ Page (8KB)               │
├──────────────────────────┤
│ Row 1                    │
│ Row 2                    │
│ Row 3                    │
│                          │
│ [Free space: 30%]        │
│ (Reserved for UPDATEs)   │
└──────────────────────────┘
UPDATE can use free space → HOT update possible
```

**When to lower fillfactor:**
- Tables with frequent UPDATEs (especially on indexed columns)
- Enables more HOT (Heap-Only Tuple)[^5] updates
- Reduces bloat from UPDATE operations

**When to keep fillfactor = 100:**
- Insert-only tables (no UPDATEs)
- Read-mostly tables
- Need maximum storage efficiency

**Example:**
```sql
-- Create table with 70% fillfactor (30% free space)
CREATE TABLE frequently_updated (
    id SERIAL PRIMARY KEY,
    status TEXT,
    updated_at TIMESTAMP
) WITH (fillfactor = 70);

-- Alter existing table
ALTER TABLE users SET (fillfactor = 80);

-- Check current fillfactor
SELECT relname, reloptions
FROM pg_class
WHERE relname = 'users';
```

## Why it matters

### Storage Capacity Planning

**Understanding data growth:**
- Monitor `PGDATA` size and growth rate
- Track individual database sizes
- Identify bloated tables (dead tuples, TOAST bloat)
- Plan for VACUUM space reclamation

**Filesystem capacity:**
- PostgreSQL requires free space for WAL files (can grow rapidly)
- VACUUM needs free space (cannot reclaim if disk 100% full)
- Checkpoints write dirty pages (temporary disk I/O spikes)

### Performance Implications

**Tablespaces for I/O separation:**
- Place WAL on separate fast storage (reduce checkpoint impact)
- Separate hot tables to SSD, archives to HDD
- Reduce I/O contention between workloads

**TOAST overhead:**
- Out-of-line TOAST data requires extra I/O to fetch
- Frequently accessing TOASTed columns = performance penalty
- Consider column width and access patterns

**fillfactor tuning:**
- Lower fillfactor = more HOT updates = better performance
- But: lower fillfactor = more disk space usage
- Trade-off between space efficiency and update performance

### Operational Reliability

**Disk full scenarios:**
- PostgreSQL cannot start if disk 100% full
- Writes fail, database becomes read-only
- VACUUM cannot run (bloat cannot be cleaned)
- WAL files cannot be created (crashes)

**Backup considerations:**
- Need enough free space for pg_basebackup
- Logical backups (pg_dump) write to disk
- PITR requires WAL archiving (disk space for archive_command)

## How to monitor

### Check Database Sizes

```sql
-- List all databases with sizes
SELECT
    datname AS database_name,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

**Example output:**
```
 database_name | size
---------------+-------
 production_db | 450 GB
 staging_db    | 120 GB
 postgres      | 8 MB
 template1     | 8 MB
 template0     | 8 MB
```

### Check Table and Index Sizes

```sql
-- Top 10 largest tables with indexes
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS indexes_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

**Example output:**
```
 schemaname | tablename      | total_size | table_size | indexes_size
------------+----------------+------------+------------+-------------
 public     | events         | 85 GB      | 60 GB      | 25 GB
 public     | transactions   | 42 GB      | 30 GB      | 12 GB
 public     | user_activity  | 28 GB      | 20 GB      | 8 GB
```

**What to look for:**
- Tables with indexes larger than table itself (over-indexing)
- Unexpectedly large tables (possible bloat)
- TOAST tables (`pg_toast.*`) appearing in top list (TOAST bloat)

### Check TOAST Table Sizes

```sql
-- Find TOAST tables and their sizes
SELECT
    n.nspname AS schema,
    c.relname AS table_name,
    c2.relname AS toast_table,
    pg_size_pretty(pg_total_relation_size(c2.oid)) AS toast_size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
JOIN pg_class c2 ON c2.oid = c.reltoastrelid
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
  AND pg_total_relation_size(c2.oid) > 1024 * 1024 * 100  -- > 100MB
ORDER BY pg_total_relation_size(c2.oid) DESC
LIMIT 10;
```

**Example output:**
```
 schema | table_name | toast_table        | toast_size
--------+------------+--------------------+-----------
 public | documents  | pg_toast_16432     | 12 GB
 public | audit_logs | pg_toast_16445     | 8 GB
```

**What to look for:**
- TOAST tables larger than expected (may need VACUUM)
- Frequently accessed tables with large TOAST (performance impact)

### Check Tablespace Usage

```sql
-- List all tablespaces and their locations
SELECT
    spcname AS tablespace_name,
    pg_tablespace_location(oid) AS location
FROM pg_tablespace;

-- Objects in each tablespace
SELECT
    ts.spcname AS tablespace,
    n.nspname AS schema,
    c.relname AS object_name,
    c.relkind AS type,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
JOIN pg_tablespace ts ON ts.oid = c.reltablespace
WHERE c.relkind IN ('r', 'i')  -- tables and indexes
ORDER BY pg_total_relation_size(c.oid) DESC
LIMIT 20;
```

### Monitor Disk Space (OS Level)

```bash
# Check PGDATA filesystem usage
df -h /var/lib/postgresql/data

# Check pg_wal size
du -sh /var/lib/postgresql/data/pg_wal

# Find largest files in PGDATA
du -h /var/lib/postgresql/data/* | sort -rh | head -20

# Check inode usage (can run out even with free space)
df -i /var/lib/postgresql/data
```

**Example output:**
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       500G  380G  120G  76% /var/lib/postgresql/data

/var/lib/postgresql/data/pg_wal: 2.1G
```

**What to look for:**
- Disk usage > 80%: Start planning for expansion
- Disk usage > 90%: Critical - VACUUM may fail
- pg_wal suddenly large: Check replication slots, archiving
- Inode usage high: Too many small files

### Check fillfactor Settings

```sql
-- List tables with custom fillfactor
SELECT
    n.nspname AS schema,
    c.relname AS table_name,
    c.reloptions AS options
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND c.reloptions IS NOT NULL
  AND c.reloptions::text LIKE '%fillfactor%';
```

## Common problems

### Problem: Disk full, PostgreSQL cannot start

**Symptom**: Server won't start, errors about "no space left on device"

**Diagnosis:**
```bash
# Check disk usage
df -h /var/lib/postgresql/data

# Find what's using space
du -sh /var/lib/postgresql/data/* | sort -rh
```

**Solutions:**
1. **Free space immediately**:
   ```bash
   # Remove old WAL files (if safe, no archiving or replication)
   # ONLY if you understand the risks - can break replication and LOOSE data

   # Safer: Archive and remove old log files
   gzip /var/log/postgresql/*.log
   rm /var/log/postgresql/*.log.gz.1

   # Check for temp files
   rm -rf /var/lib/postgresql/data/pgsql_tmp/*
   ```

2. **Drop inactive replication slots**:
   ```sql
   SELECT pg_drop_replication_slot('inactive_slot_name');
   ```

3. **Expand filesystem** or add new volume

4. **Move tablespace to different volume**:
   ```sql
   ALTER TABLESPACE old_tablespace RENAME TO old_tablespace_backup;
   CREATE TABLESPACE new_tablespace LOCATION '/mnt/new_volume';
   ALTER TABLE large_table SET TABLESPACE new_tablespace;
   ```

### Problem: Table size growing despite deletes

**Symptom**: DELETE removes rows but table size doesn't decrease

**Cause**: Dead tuples and table bloat - VACUUM marks space as reusable but doesn't return it to OS (by design)

**See**: [Chapter 3: MVCC and Transactions - Monitor Dead Tuples and Bloat](03-mvcc-transactions.md#monitor-dead-tuples-and-bloat) for detailed diagnosis queries and solutions including:
- Checking dead tuple counts
- VACUUM vs VACUUM FULL
- pg_repack for online bloat removal
- Autovacuum tuning

### Problem: TOAST table bloat

**Symptom**: TOAST tables consuming excessive disk space

**Diagnosis:**
```sql
-- Check TOAST bloat
SELECT
    n.nspname AS schema,
    c.relname AS table_name,
    pg_size_pretty(pg_relation_size(c.oid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(c.reltoastrelid)) AS toast_size,
    ROUND(100.0 * pg_total_relation_size(c.reltoastrelid) /
          NULLIF(pg_total_relation_size(c.oid), 0), 2) AS toast_pct
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND c.reltoastrelid != 0
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(c.reltoastrelid) DESC
LIMIT 10;
```

**Causes:**
- Frequent updates to large columns
- VACUUM not keeping up with TOAST table dead tuples
- No VACUUM on TOAST tables (autovacuum should handle, but may lag)

**Solutions:**
1. **VACUUM main table** (also vacuums TOAST):
   ```sql
   VACUUM VERBOSE table_name;
   ```

2. **Adjust TOAST storage strategy**:
   ```sql
   -- For pre-compressed data (images, compressed text)
   ALTER TABLE documents ALTER COLUMN file_data SET STORAGE EXTERNAL;
   ```

3. **Consider column redesign**:
   - Move large rarely-accessed columns to separate table
   - Store large objects outside database (object storage like S3)

### Problem: Slow queries due to TOAST fetches

**Symptom**: Queries slow when selecting large columns

**Diagnosis:**
```sql
-- Find tables with TOASTed columns
SELECT
    n.nspname AS schema,
    c.relname AS table_name,
    a.attname AS column_name,
    t.typname AS type,
    a.attstorage AS storage
FROM pg_attribute a
JOIN pg_class c ON c.oid = a.attrelid
JOIN pg_namespace n ON n.oid = c.relnamespace
JOIN pg_type t ON t.oid = a.atttypid
WHERE a.attnum > 0
  AND NOT a.attisdropped
  AND c.relkind = 'r'
  AND a.attstorage != 'p'  -- Not PLAIN
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, c.relname, a.attnum;
```

**Solutions:**
1. **SELECT only needed columns** (avoid SELECT *):
   ```sql
   -- Bad: Fetches all TOASTed columns
   SELECT * FROM documents WHERE id = 123;

   -- Good: Only fetches needed columns
   SELECT id, title, created_at FROM documents WHERE id = 123;
   ```

2. **Move large columns to separate table**:
   ```sql
   -- Split into two tables
   CREATE TABLE documents_metadata (
       id SERIAL PRIMARY KEY,
       title TEXT,
       created_at TIMESTAMP
   );

   CREATE TABLE documents_content (
       id INT PRIMARY KEY REFERENCES documents_metadata(id),
       content TEXT,
       attachments BYTEA
   );
   ```

3. **Use TOAST MAIN storage** for frequently accessed small text:
   ```sql
   ALTER TABLE table_name ALTER COLUMN column_name SET STORAGE MAIN;
   ```

### Problem: Tablespace on wrong disk

**Symptom**: Slow performance, realized tablespace on slow disk

**Solution**: Move tablespace to different location

```sql
-- 1. Create new tablespace on faster disk
CREATE TABLESPACE new_fast_storage LOCATION '/mnt/nvme/pg_data';

-- 2. Move tables to new tablespace
ALTER TABLE table1 SET TABLESPACE new_fast_storage;
ALTER TABLE table2 SET TABLESPACE new_fast_storage;

-- 3. Move indexes
ALTER INDEX index1 SET TABLESPACE new_fast_storage;

-- 4. Set new default for database
ALTER DATABASE mydb SET TABLESPACE new_fast_storage;

-- 5. Drop old tablespace (after moving all objects)
DROP TABLESPACE old_slow_storage;
```

**Note**: Moving tablespaces requires exclusive locks and can take time for large tables.

## Additional Resources

1. [PostgreSQL Documentation: Database File Layout](https://www.postgresql.org/docs/current/storage-file-layout.html)
2. [PostgreSQL Documentation: TOAST](https://www.postgresql.org/docs/current/storage-toast.html)
3. [PostgreSQL Documentation: Tablespaces](https://www.postgresql.org/docs/current/manage-ag-tablespaces.html)
4. [PostgreSQL Source: Free Space Map Implementation](https://github.com/postgres/postgres/blob/master/src/backend/storage/freespace/README)
5. [PostgreSQL Wiki: pg_repack](https://github.com/reorg/pg_repack)

[^1]: [PostgreSQL Documentation: Database Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html)
[^2]: [PostgreSQL Documentation: Free Space Map](https://www.postgresql.org/docs/current/storage-fsm.html)
[^3]: [PostgreSQL Documentation: Visibility Map](https://www.postgresql.org/docs/current/storage-vm.html)
[^4]: [PostgreSQL Documentation: Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
[^5]: [PostgreSQL Documentation: Heap-Only Tuples (HOT)](https://www.postgresql.org/docs/current/storage-hot.html)
