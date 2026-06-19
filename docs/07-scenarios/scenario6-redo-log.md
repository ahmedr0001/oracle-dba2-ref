---
tags: [scenario, redo-log-loss, clear-logfile, current, active, inactive]
---

# Scenario 6 — Redo Log Group Loss

**Situation:** One or more online redo log group members (or an entire group) are lost or corrupted. What you do depends entirely on the **status** of the affected group.

---

## Step 0 — Diagnose the Situation First

```sql
-- Check which groups exist and their current status
SELECT GROUP#, SEQUENCE#, STATUS, ARCHIVED
FROM V$LOG
ORDER BY GROUP#;

-- Check the physical member files
SELECT GROUP#, MEMBER, STATUS
FROM V$LOGFILE
ORDER BY GROUP#;
```

| Status | Archived | What You'll Lose | Fix |
|---|---|---|---|
| INACTIVE | YES | Nothing | `CLEAR LOGFILE` |
| INACTIVE | NO | Nothing | `CLEAR UNARCHIVED LOGFILE` + backup immediately |
| ACTIVE | YES | Possible data loss | Restore + `RECOVER UNTIL CANCEL` |
| ACTIVE | NO | Data loss | Restore + `RECOVER UNTIL CANCEL` |
| CURRENT | — | Unarchived transactions | Restore + `RECOVER DATABASE` + RESETLOGS |

---

## Case 1 — INACTIVE + Archived (Safest)

The group has already been archived and checkpointed. No data at risk. Oracle recreates the member files automatically.

```sql
-- Clear by group number — Oracle recreates empty member files
ALTER DATABASE CLEAR LOGFILE GROUP 3;

-- Or clear by file path
ALTER DATABASE CLEAR LOGFILE '/u01/oradata/ORCL/redo03a.log';

-- Verify the group is back
SELECT GROUP#, STATUS, ARCHIVED FROM V$LOG;
```

---

## Case 2 — INACTIVE + NOT Archived

The data in this group was never archived, but since it's INACTIVE it has already been written (checkpointed) to the datafiles. Clearing it creates a gap in the archive log chain — **take a full backup immediately after**.

```sql
-- UNARCHIVED keyword is required when the log was never archived
ALTER DATABASE CLEAR UNARCHIVED LOGFILE GROUP 3;

-- Verify
SELECT GROUP#, STATUS, ARCHIVED FROM V$LOG;
```

```bash
# Take a full backup immediately — the archive log chain is now broken
# Old backups may not be recoverable past this point
rman target /
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
```

---

## Case 3 — ACTIVE + Archived (May Cause Data Loss)

ACTIVE means Oracle hasn't finished the checkpoint yet — the dirty blocks are not fully written to the datafiles. Losing this log means instance recovery cannot complete.

```sql
-- Try a checkpoint first to clear ACTIVE status (may work if disk isn't gone)
ALTER SYSTEM CHECKPOINT;

-- Check if it moved to INACTIVE
SELECT GROUP#, STATUS FROM V$LOG;
```

If the checkpoint fails (disk is truly gone):

```sql
SHUTDOWN IMMEDIATE;
```

```bash
rman target /
```

```sql
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;

-- Cancel-based recovery: apply archive logs, stop when no more are available
RMAN> RECOVER DATABASE UNTIL CANCEL;
-- Or from SQL*Plus: SQL> RECOVER DATABASE UNTIL CANCEL;
-- Type CANCEL at the prompt when the missing log is requested

-- Must use RESETLOGS because recovery is incomplete
SQL> ALTER DATABASE OPEN RESETLOGS;
```

---

## Case 4 — CURRENT Group Lost (Most Serious)

The database was actively writing to this log. Unarchived transactions in this log are at risk.

```sql
-- If DB is still up, try a clean shutdown
SHUTDOWN IMMEDIATE;
-- If DB is hung/crashed, you may already be in MOUNT state
```

**Option A — Try complete recovery (if the log was archived before loss):**

```bash
rman target /
```

```sql
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;          -- applies all available archive logs
SQL> ALTER DATABASE OPEN RESETLOGS;   -- RESETLOGS creates new redo log members
```

**Option B — If log was NOT archived (data loss is unavoidable):**

```sql
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL CANCEL;  -- apply as much as possible, then cancel
SQL> ALTER DATABASE OPEN RESETLOGS;   -- accept the data loss and open
```

!!! warning "OPEN RESETLOGS discards the current log"
    Any committed transactions that were only in the lost CURRENT log and never archived are permanently gone. `OPEN RESETLOGS` accepts this and starts fresh.

---

## Pre-Emptive: Force Log Switch to Avoid Losing CURRENT

```sql
-- Switch the current log to move it to ACTIVE, then INACTIVE
ALTER SYSTEM SWITCH LOGFILE;

-- Force checkpoint to speed the transition from ACTIVE → INACTIVE
ALTER SYSTEM CHECKPOINT;

-- Confirm the group you wanted to clear is now INACTIVE
SELECT GROUP#, STATUS, ARCHIVED FROM V$LOG;
```

---

## Section Commands Summary

```sql
-- Diagnose
SELECT GROUP#, SEQUENCE#, STATUS, ARCHIVED FROM V$LOG;
SELECT GROUP#, MEMBER, STATUS FROM V$LOGFILE;

-- Case 1: INACTIVE + Archived (no risk)
ALTER DATABASE CLEAR LOGFILE GROUP 3;
ALTER DATABASE CLEAR LOGFILE '/path/redo03.log';

-- Case 2: INACTIVE + NOT Archived (no data loss, but re-backup immediately)
ALTER DATABASE CLEAR UNARCHIVED LOGFILE GROUP 3;
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;

-- Case 3 & 4: ACTIVE or CURRENT (restore required)
SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL CANCEL;   -- cancel-based (for ACTIVE/missing logs)
SQL> ALTER DATABASE OPEN RESETLOGS;

-- Case 4 complete recovery attempt (if log was archived)
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
SQL> ALTER DATABASE OPEN RESETLOGS;

-- Force log switch + checkpoint
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM CHECKPOINT;
```
