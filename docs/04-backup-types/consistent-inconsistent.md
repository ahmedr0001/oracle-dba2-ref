---
tags: [backup, consistent, inconsistent, cold, hot, offline, online]
---

# Consistent vs Inconsistent Backup

Every RMAN backup is either **consistent** or **inconsistent** — this tells Oracle whether it needs archive logs to make the restored files usable.

| | Consistent (Cold/Offline) | Inconsistent (Hot/Online) |
|---|---|---|
| DB state during backup | MOUNT (cleanly shut down) | OPEN (database running) |
| Datafile headers | All at the same SCN | Different SCNs |
| Archive logs needed to recover? | No | Yes |
| ARCHIVELOG mode required? | No | Yes |

---

## Consistent Backup (Cold / Offline)

Database is cleanly shut down and mounted. All datafile headers agree on the same checkpoint SCN — the backup can be opened directly without applying any archive logs.

```sql
-- Step 1: clean shutdown — must NOT use ABORT
-- (ABORT leaves the DB in a dirty state; headers won't match)
SHUTDOWN IMMEDIATE;

-- Step 2: start to MOUNT only — do not open
STARTUP MOUNT;

-- Step 3: back up while closed
RMAN> BACKUP DATABASE;

-- Step 4: open the database when done
ALTER DATABASE OPEN;
```

!!! warning "SHUTDOWN ABORT = inconsistent"
    If you use `SHUTDOWN ABORT`, the DB is not cleanly shut down. A backup taken after ABORT (even from MOUNT) is still inconsistent — instance recovery is pending. Always use `SHUTDOWN IMMEDIATE` for cold backups.

---

## Inconsistent Backup (Hot / Online)

Database is open and serving users. Datafile headers have different SCNs because Oracle is writing continuously. This is perfectly normal — RMAN records the SCNs, and archive logs fill the gap during recovery.

```sql
-- Database must be in ARCHIVELOG mode for hot backups
SELECT LOG_MODE FROM V$DATABASE;  -- must show ARCHIVELOG

-- Back up while open (online/hot backup)
RMAN> BACKUP DATABASE;

-- Hot backup + archive logs (self-contained — recommended)
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
```

---

## Validity Matrix

The most important table — memorize this:

| Archive Mode | DB State at Backup | Backup Type | Valid? |
|---|---|---|---|
| `ARCHIVELOG` | OPEN | Inconsistent (hot) | ✅ Valid — need archive logs at recovery |
| `ARCHIVELOG` | MOUNT (closed) | Consistent (cold) | ✅ Valid — no archive logs needed |
| `NOARCHIVELOG` | MOUNT (closed) | Consistent (cold) | ✅ Valid — no archive logs needed |
| `NOARCHIVELOG` | OPEN | Inconsistent (hot) | ❌ **Invalid — RMAN will refuse** |

---

## Recovery Steps for Each Type

=== "Restore Consistent Backup"
    ```sql
    -- Restore the datafiles from backup
    RMAN> RESTORE DATABASE;

    -- Open directly — SCNs already match, no recovery needed
    RMAN> ALTER DATABASE OPEN;
    -- If control file was also restored:
    RMAN> ALTER DATABASE OPEN RESETLOGS;
    ```

=== "Restore Inconsistent Backup"
    ```sql
    -- Restore the datafiles
    RMAN> RESTORE DATABASE;

    -- Apply archive logs to bring all files to the same SCN
    -- RMAN automatically figures out which archive logs to apply
    RMAN> RECOVER DATABASE;

    -- Open the database
    RMAN> ALTER DATABASE OPEN;
    -- or with RESETLOGS if required:
    RMAN> ALTER DATABASE OPEN RESETLOGS;
    ```

---

## Section Commands Summary

```sql
-- Check archive mode
SELECT LOG_MODE FROM V$DATABASE;

-- Consistent (cold) backup
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
RMAN> BACKUP DATABASE;
ALTER DATABASE OPEN;

-- Inconsistent (hot) backup (ARCHIVELOG mode required)
RMAN> BACKUP DATABASE;
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;

-- Restore consistent
RMAN> RESTORE DATABASE;
RMAN> ALTER DATABASE OPEN;

-- Restore inconsistent
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN;
RMAN> ALTER DATABASE OPEN RESETLOGS;  -- use when required after restore
```
