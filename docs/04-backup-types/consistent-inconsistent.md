---
tags: [backup, consistent, inconsistent, cold, hot, offline, online]
---

# Consistent vs Inconsistent Backup

## The Core Concept

Every Oracle backup is either **consistent** or **inconsistent** — this refers to whether all datafile headers contain the same SCN (System Change Number) at the time of backup.

| Type | Also Called | DB State | Archive Logs to Recover? |
|---|---|---|---|
| **Consistent** | Cold / Offline | Cleanly shut down (MOUNT) | ❌ Not needed |
| **Inconsistent** | Hot / Online | Open (OPEN) | ✅ Required |

---

## Consistent Backup (Cold / Offline)

A consistent backup is taken when the database is in **MOUNT** state — cleanly shut down. All datafile headers agree on the same checkpoint SCN. The backup can be restored and immediately opened without applying any archive logs.

```sql
-- Step 1: Clean shutdown (NOT ABORT)
SHUTDOWN IMMEDIATE;

-- Step 2: Mount only (do not open)
STARTUP MOUNT;

-- Step 3: Take the consistent backup
RMAN> BACKUP DATABASE;

-- Step 4: Open the database
ALTER DATABASE OPEN;
```

**When to use consistent backups:**
- Database is in `NOARCHIVELOG` mode (your only RMAN option)
- Zero-risk baseline before a major migration or upgrade
- Dev/test databases where downtime is acceptable

!!! warning "SHUTDOWN ABORT = inconsistent state"
    If you use `SHUTDOWN ABORT`, the database is NOT cleanly shut down — datafiles have different SCNs. A backup taken from MOUNT after ABORT is still considered inconsistent because instance recovery is still pending. Always use `SHUTDOWN IMMEDIATE` for consistent cold backups.

---

## Inconsistent Backup (Hot / Online)

An inconsistent backup is taken while the database is **open** and serving users. Datafile headers contain different SCNs because Oracle is continuously writing. This is normal and expected — RMAN records the SCNs, and archive logs fill the gap during recovery.

```sql
-- Database must be in ARCHIVELOG mode for hot backups
-- Check: SELECT LOG_MODE FROM V$DATABASE;

-- Take backup while database is open (online)
RMAN> BACKUP DATABASE;                    -- inconsistent but valid
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;    -- inconsistent but self-contained
```

**When hot backup is valid:**
- Database is in `ARCHIVELOG` mode ✅
- All archive logs from backup time to recovery time are available ✅

---

## The Validity Matrix

This is the key table to memorize — what combinations are valid:

| Database Mode | DB State at Backup | Backup Type | Valid? |
|---|---|---|---|
| `ARCHIVELOG` | OPEN | Inconsistent | ✅ Valid — need archive logs to recover |
| `ARCHIVELOG` | MOUNT (closed) | Consistent | ✅ Valid — no archive logs needed |
| `NOARCHIVELOG` | MOUNT (closed) | Consistent | ✅ Valid — no archive logs needed |
| `NOARCHIVELOG` | OPEN | Inconsistent | ❌ **Invalid** — cannot recover without archive logs |

!!! danger "NOARCHIVELOG + Open = Invalid backup"
    RMAN will actually refuse to back up an open database in NOARCHIVELOG mode. Even if you could force it, you'd have no way to make the backup consistent on restore — the archive logs don't exist.

---

## Recovery from Each Type

=== "Restore a Consistent Backup"
    ```sql
    -- 1. Restore the datafiles
    RMAN> RESTORE DATABASE;

    -- 2. Open directly — no recovery needed (SCNs already match)
    RMAN> ALTER DATABASE OPEN;
    -- If control file was also restored:
    RMAN> ALTER DATABASE OPEN RESETLOGS;
    ```

=== "Restore an Inconsistent Backup"
    ```sql
    -- 1. Restore the datafiles
    RMAN> RESTORE DATABASE;

    -- 2. Apply archive logs to bring to consistent state
    RMAN> RECOVER DATABASE;
    -- (RMAN automatically applies the right archive logs)

    -- 3. Open the database
    RMAN> ALTER DATABASE OPEN;
    -- or if using RESETLOGS:
    RMAN> ALTER DATABASE OPEN RESETLOGS;
    ```

---

## How RMAN Handles the Inconsistency

RMAN knows how to deal with inconsistent backups automatically:

1. **During backup:** RMAN records the start SCN and end SCN of every datafile
2. **During restore:** RMAN restores the files to their backup-time state
3. **During recovery:** RMAN applies archive logs (and online redo if needed) to advance all datafiles to the same SCN — making them consistent
4. **On open:** The database performs instance recovery if needed, then opens

```
Backup taken at SCN 5000 (datafiles at various SCNs 4800–5000)
Archive logs: SCN 5000 → SCN 8000 (recovery target)

Restore ──────► Apply archive logs ──────► Open
(SCN ~5000)    (advance to SCN 8000)     (consistent)
```

---

## Choosing Your Backup Mode

```
Decision tree:
──────────────
Is ARCHIVELOG mode enabled?
  │
  ├── NO  ──► You MUST take consistent (cold) backups only
  │          SHUTDOWN IMMEDIATE → STARTUP MOUNT → BACKUP → OPEN
  │
  └── YES ──► You can take both consistent AND inconsistent backups
              ├── During business hours: hot backup (OPEN) ← preferred
              └── Maintenance window: cold backup (MOUNT) ← optional
```
