# PostgreSQL Core Concepts and Operations Guide

## Purpose

This guide is designed to help platform engineers, software developers, Kubernetes specialists, and network engineers understand the essential PostgreSQL concepts and architecture that affect day-to-day operations.

## Target Audience

- Platform engineers
- Kubernetes/Infrastructure specialists
- Software developers
- Network engineers
- Anyone operating PostgreSQL in production environments

## Topics

### 1. [Process Architecture](01-process-architecture.md)
Understanding PostgreSQL's process model, backend processes, auxiliary processes, and connection management.

### 2. [Memory Management](02-memory-management.md)
Shared buffers, work memory, maintenance work memory, and how PostgreSQL uses memory.

### 3. [MVCC and Transactions](03-mvcc-transactions.md)
Multi-Version Concurrency Control, transaction isolation levels, visibility, VACUUM, autovacuum, and bloat management.

### 4. [WAL (Write-Ahead Log)](04-wal.md)
Write-Ahead Logging, checkpoints, durability, and crash recovery.

### 5. [Replication](05-replication.md)
Streaming replication, replication slots, synchronous vs asynchronous replication, failover concepts.

### 6. [Tablespaces and Storage](06-tablespaces-storage.md)
Physical layout, TOAST, fillfactor, and storage management.

### 7. [Critical Monitoring](07-monitoring.md)
Essential views, key metrics, and logging configuration for production operations.

### 8. [Backup and Recovery](08-backup-recovery.md)
Backup strategies, PITR, modern tools, and disaster recovery concepts.

### 9. [Upgrade and Maintenance](09-upgrade-maintenance.md)
pg_upgrade, logical replication for upgrades, extension management, and version policies.

### 10. [Configuration and Tuning](10-configuration-tuning.md)
Critical postgresql.conf parameters, authentication, and workload-specific tuning.

### 11. [Common Troubleshooting](11-troubleshooting.md)
Practical solutions for replication lag, connection issues, and disk space problems.

### 12. [Operational Security](12-security.md)
Roles, privileges, SSL/TLS, audit logging, and row-level security.

### 13. [TimescaleDB Specific](13-timescaledb.md)
Hypertables, chunks, compression, continuous aggregates, and operational considerations.

## How to Use This Guide

Each topic is structured as follows:
- **What is it?** - Core concepts explained
- **Why it matters** - Operational impact and importance
- **How to monitor** - Practical queries and metrics
- **Common problems** - Symptoms and solutions
- **References** - Official documentation and resources

Start with topics relevant to your current challenges, or read sequentially for comprehensive understanding.

## Contributing

This is a living document. If you find gaps, errors, or have suggestions for improvement, please contribute.
