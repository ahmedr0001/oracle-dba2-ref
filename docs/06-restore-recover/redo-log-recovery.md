---
tags: [redo-log, recovery, clear-logfile, active, inactive]
---

# Redo Log Recovery

Losing redo log groups is more complex than losing datafiles — what you do depends on the **status** of the lost group and whether it has been **archived**.

---

## Decision: What Status Is the Lost Group?

```sql
-- First, check all group statuses
SELECT GROUP#, SEQUENCE#, STATUS, ARCHIVED FROM V$LOG;
SELECT GROUP#, MEMBER, STATUS FROM V$LOGFILE;
```

| Group Status | Archived? | Risk | Fix |
|---|---|---|---|
| `INACTIVE` | Yes | None | `ALTER DATABASE CLEAR LOGFILE` |
| `INACTIVE` | No | None | `ALTER DATABASE CLEAR UNARCHIVED LOGFILE` |
| `ACTIVE` | Yes | Possible data loss | Restore + `RECOVER DATABASE UNTIL CANCEL` |
| `ACTIVE` | No | Data loss | Restore + `RECOVER DATABASE UNTIL CANCEL` |
| `CURRENT` | — | **Serious** — instance crash | Restore + `RECOVER DATABASE` then `RESETLOGS` |

---

## Case 1: INACTIVE Group — Archived

No data at risk. Just clear the log group to recreate its member files.

```sql
-- Clear a log group by group number (Oracle recreates the member files)
ALTER DATABASE CLEAR LOGFILE GROUP 3;

-- Or clear by file path
ALTER DATABASE CLEAR LOGFILE '/u01/oradata/ORCL/redo03a.log';
```

---

## Case 2: INACTIVE Group — NOT Archived

Data in the group was never archived. Clearing it means that data is unrecoverable — but since it's INACTIVE, it has already been checkpointed to datafiles, so no actual data loss occurs.

```sql
-- Clear unarchived inactive log (UNARCHIVED keyword required)
ALTER DATABASE CLEAR UNARCHIVED LOGFILE GROUP 3;

-- After clearing, take a full backup immediately
-- (archive log chain is now broken; old backups may not be restorable)
RMAN> BACKUP DATABASE;
```

---

## Case 3: ACTIVE Group (Archived or Not)

ACTIVE means Oracle hasn't finished checkpointing it yet. Losing it means a crash recovery problem. You must restore from backup.

```sql
-- Step 1: shutdown if not already down
SHUTDOWN IMMEDIATE;

-- Step 2: restore the whole database from the last valid backup
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;

-- Step 3: recover using cancel-based recovery (apply logs until you can't)
-- Type CANCEL when prompted after the last archive log that's available
RMAN> RECOVER DATABASE UNTIL CANCEL;
-- Or SQL: RECOVER DATABASE UNTIL CANCEL;

-- Step 4: open with RESETLOGS (required after cancel-based recovery)
SQL> ALTER DATABASE OPEN RESETLOGS;
```

---

## Case 4: CURRENT Group (DB Was Open)

The database was actively writing to this group. This is the most serious case — the current log may contain uncommitted work not yet in the datafiles.

```sql
-- If instance is still running, try a clean shutdown first
SHUTDOWN IMMEDIATE;

-- Restore all datafiles
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;

-- Apply all archive logs — RMAN stops when it can't find more
RMAN> RECOVER DATABASE;

-- Open with RESETLOGS — Oracle creates new redo log members
SQL> ALTER DATABASE OPEN RESETLOGS;
```

!!! warning "CURRENT group loss with no archive logs = data loss"
    If the CURRENT group was not yet archived and the disk holding it is gone, the transactions in that log are permanently lost. `OPEN RESETLOGS` discards them.

---

## Case 5: ACTIVE + Archived (May Cause Data Loss)

```sql
SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL CANCEL;
SQL> ALTER DATABASE OPEN RESETLOGS;
```

---

## Force a Log Switch + Checkpoint (Helpful Before Maintenance)

```sql
-- Switch the current log to move it to ACTIVE then INACTIVE
ALTER SYSTEM SWITCH LOGFILE;

-- Force checkpoint to move ACTIVE → INACTIVE faster
ALTER SYSTEM CHECKPOINT;

-- Verify the group is now INACTIVE before clearing it
SELECT GROUP#, STATUS, ARCHIVED FROM V$LOG;
```

---

## Section Commands Summary

```sql
-- Check redo log status
SELECT GROUP#, SEQUENCE#, STATUS, ARCHIVED FROM V$LOG;
SELECT GROUP#, MEMBER, STATUS FROM V$LOGFILE;

-- Clear inactive archived group
ALTER DATABASE CLEAR LOGFILE GROUP 3;
ALTER DATABASE CLEAR LOGFILE '/path/redo03.log';

-- Clear inactive NOT archived (no data loss, but back up immediately after)
ALTER DATABASE CLEAR UNARCHIVED LOGFILE GROUP 3;
RMAN> BACKUP DATABASE;

-- Active or Current group loss: restore + cancel-based recovery
SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL CANCEL;   -- or: SQL> RECOVER DATABASE UNTIL CANCEL;
SQL> ALTER DATABASE OPEN RESETLOGS;

-- Current group (complete recovery attempt)
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
SQL> ALTER DATABASE OPEN RESETLOGS;

-- Force log switch and checkpoint
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM CHECKPOINT;
```
