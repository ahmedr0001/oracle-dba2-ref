---
tags: [restore, recover]
---

# Restore vs Recover

These two words mean very different things in Oracle — using the wrong one in a scenario answer will cost you marks.

| | RESTORE | RECOVER |
|---|---|---|
| **What it does** | Copies backup files back to disk | Applies redo (archive logs + online redo) to bring files forward |
| **Input** | RMAN backup sets or image copies | Archive logs and online redo logs |
| **Output** | Datafiles at the SCN of the backup | Datafiles at the current SCN (or a target point) |
| **DB state** | MOUNT (usually) | MOUNT (usually) |
| **Alone enough?** | No — data is stale from backup time | No — needs restored files to apply changes to |

```
Timeline:
─────────────────────────────────────────────────────►
   Backup taken           Changes happened     Now/failure
       │                        │                  │
  [RESTORE]──────────────►[RECOVER]─────────────►[OPEN]
  (put old files back)   (apply archive logs)
```

---

## Basic Commands

```sql
-- RESTORE: put files back from backup
RMAN> RESTORE DATABASE;           -- all datafiles
RMAN> RESTORE TABLESPACE users;   -- one tablespace
RMAN> RESTORE DATAFILE 4;         -- one datafile
RMAN> RESTORE CONTROLFILE FROM AUTOBACKUP;  -- control file

-- RECOVER: apply redo to bring restored files to consistent state
RMAN> RECOVER DATABASE;           -- apply logs to bring DB to current SCN
RMAN> RECOVER TABLESPACE users;   -- apply logs to one tablespace
RMAN> RECOVER DATAFILE 4;         -- apply logs to one datafile

-- Full sequence (complete recovery)
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
SQL>  ALTER DATABASE OPEN;
```

---

## What RECOVER Actually Does

1. Finds the SCN recorded in each restored datafile header (from backup time)
2. Locates archive logs that cover from that SCN forward
3. Applies each log in sequence, replaying every committed transaction
4. Applies online redo logs if needed (for complete recovery)
5. All datafiles converge to the same SCN → database is consistent → can open

---

## Section Commands Summary

```sql
-- Restore (put files back from backup)
RMAN> RESTORE DATABASE;
RMAN> RESTORE TABLESPACE users;
RMAN> RESTORE DATAFILE 4;
RMAN> RESTORE CONTROLFILE FROM AUTOBACKUP;
RMAN> RESTORE CONTROLFILE FROM '/path/backup.bkp';

-- Recover (apply archive logs to bring forward)
RMAN> RECOVER DATABASE;
RMAN> RECOVER TABLESPACE users;
RMAN> RECOVER DATAFILE 4;

-- Open after recovery
SQL> ALTER DATABASE OPEN;
SQL> ALTER DATABASE OPEN RESETLOGS;  -- required after incomplete recovery
```
