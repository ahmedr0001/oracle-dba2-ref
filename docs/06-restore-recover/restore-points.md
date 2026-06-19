---
tags: [restore-point, flashback, pitr, scn]
---

# Restore Points

A restore point is a **named alias for an SCN** (or a point in time). Instead of remembering SCN numbers or exact timestamps, you create a named marker before a risky operation and recover to that name.

Used in:
- Incomplete recovery (`RECOVER DATABASE UNTIL RESTORE POINT ...`)
- Flashback Database (`FLASHBACK DATABASE TO RESTORE POINT ...`)
- Flashback Table

---

## Create a Restore Point

```sql
-- Normal restore point: stores the current SCN with a name
-- Does NOT guarantee Flashback Database capability
CREATE RESTORE POINT before_upgrade;

-- Named with a specific time reference (Oracle records current SCN)
CREATE RESTORE POINT before_batch_purge;

-- Guaranteed restore point: enables Flashback Database to this exact point
-- Oracle retains flashback logs until this point is explicitly dropped
-- Requires Flashback Database to be enabled
CREATE RESTORE POINT before_upgrade GUARANTEE FLASHBACK DATABASE;
```

!!! warning "GUARANTEE FLASHBACK DATABASE"
    This type consumes significant space in the FRA — flashback logs are kept until you manually drop the restore point. Always drop it when no longer needed.

---

## Use Restore Points in Recovery

```sql
-- Incomplete recovery to a restore point
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL RESTORE POINT before_upgrade;
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- Flashback Database to a restore point (much faster than RMAN restore)
SQL> SHUTDOWN IMMEDIATE;
SQL> STARTUP MOUNT;
SQL> FLASHBACK DATABASE TO RESTORE POINT before_upgrade;
SQL> ALTER DATABASE OPEN RESETLOGS;
```

---

## Manage Restore Points

```sql
-- List all current restore points
SELECT NAME, SCN, TIME, GUARANTEE_FLASHBACK_DATABASE
FROM V$RESTORE_POINT;

-- Drop a restore point when no longer needed
DROP RESTORE POINT before_upgrade;
```

---

## Section Commands Summary

```sql
-- Create
CREATE RESTORE POINT before_upgrade;
CREATE RESTORE POINT before_upgrade GUARANTEE FLASHBACK DATABASE;

-- Use in recovery
RMAN> RECOVER DATABASE UNTIL RESTORE POINT before_upgrade;
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- Use in Flashback
SQL> STARTUP MOUNT;
SQL> FLASHBACK DATABASE TO RESTORE POINT before_upgrade;
SQL> ALTER DATABASE OPEN RESETLOGS;

-- Manage
SELECT NAME, SCN, TIME, GUARANTEE_FLASHBACK_DATABASE FROM V$RESTORE_POINT;
DROP RESTORE POINT before_upgrade;
```
