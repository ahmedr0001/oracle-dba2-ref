---
tags: [multiplexing, control-file, redo-log]
---

# Multiplexing

## The Concept

**Multiplexing** means keeping multiple identical copies of a critical Oracle file on different disks simultaneously. If one disk fails, Oracle continues using the copies on the surviving disks — with zero downtime and zero data loss.

Two files must be multiplexed:

| File | Why It's Critical | Minimum Copies |
|---|---|---|
| **Control File** | Losing all copies = database cannot mount | 2 (Oracle default), 3 recommended |
| **Redo Log Members** | Losing all members of a group = data loss | 2 members per group, 3 groups recommended |

---

## Control File Multiplexing

### Check Current Control Files

```sql
-- From SQL*Plus (connected as SYSDBA)
SHOW PARAMETER CONTROL_FILES;

-- Or query the view
SELECT NAME, STATUS FROM V$CONTROLFILE;
```

### Add a Control File Copy

Oracle doesn't have an `ADD CONTROLFILE` command — you configure the parameter and copy the file at OS level:

```sql
-- Step 1: Check current location(s)
SHOW PARAMETER CONTROL_FILES;

-- Step 2: Shut down cleanly
SHUTDOWN IMMEDIATE;
```

```bash
# Step 3: Copy the control file to the new location (Linux)
cp /u01/oradata/ORCL/control01.ctl /u02/oradata/ORCL/control02.ctl
cp /u01/oradata/ORCL/control01.ctl /u03/oradata/ORCL/control03.ctl
```

```sql
-- Step 4: Update the parameter (SPFILE)
ALTER SYSTEM SET CONTROL_FILES =
  '/u01/oradata/ORCL/control01.ctl',
  '/u02/oradata/ORCL/control02.ctl',
  '/u03/oradata/ORCL/control03.ctl'
  SCOPE = SPFILE;

-- Step 5: Restart
STARTUP;

-- Step 6: Verify
SELECT NAME FROM V$CONTROLFILE;
```

!!! tip "One control file can recover the other"
    If you lose one control file, copy a surviving one to replace the lost file path, update the parameter if needed, and restart. The database will use the recovered copy.

---

## Redo Log Multiplexing

### Redo Log Concepts

```
Log Group Structure:
─────────────────────────────────────────────────────
Group 1 ──► Member 1A: /disk_a/redo01a.log  ┐
            Member 1B: /disk_b/redo01b.log  ┘ identical copies

Group 2 ──► Member 2A: /disk_a/redo02a.log  ┐
            Member 2B: /disk_b/redo02b.log  ┘ identical copies

Group 3 ──► Member 3A: /disk_a/redo03a.log  ┐
            Member 3B: /disk_b/redo03b.log  ┘ identical copies
```

- **Group** = logical redo log unit — Oracle writes to one group at a time
- **Member** = a physical redo log file — all members in a group are written simultaneously
- Oracle writes to groups in rotation: G1 → G2 → G3 → G1 → ...
- **Recommended:** 3 groups, 2 members each, on separate disks

### Check Current Redo Logs

```sql
-- View groups and their status
SELECT GROUP#, SEQUENCE#, BYTES/1024/1024 AS SIZE_MB, MEMBERS, STATUS
FROM V$LOG
ORDER BY GROUP#;

-- View members (physical files) for each group
SELECT GROUP#, MEMBER, STATUS
FROM V$LOGFILE
ORDER BY GROUP#, MEMBER;
```

**V$LOG STATUS values:**

| Status | Meaning |
|---|---|
| `CURRENT` | Oracle is actively writing to this group |
| `ACTIVE` | Used but not yet archived (needed for recovery) |
| `INACTIVE` | Archived — safe to reuse |
| `UNUSED` | Never been written to (brand new group) |

### Add a Member to an Existing Group

```sql
-- Add a second member to Group 1 (multiplexing it)
ALTER DATABASE ADD LOGFILE MEMBER
  '/u02/oradata/ORCL/redo01b.log'
  TO GROUP 1;

-- Add members to multiple groups at once
ALTER DATABASE ADD LOGFILE MEMBER
  '/u02/oradata/ORCL/redo01b.log' TO GROUP 1,
  '/u02/oradata/ORCL/redo02b.log' TO GROUP 2,
  '/u02/oradata/ORCL/redo03b.log' TO GROUP 3;
```

### Add a New Group (with Members)

```sql
-- Add a completely new group with 2 members
ALTER DATABASE ADD LOGFILE GROUP 4
  ('/u01/oradata/ORCL/redo04a.log',
   '/u02/oradata/ORCL/redo04b.log')
  SIZE 200M;
```

### Drop a Log Member

```sql
-- Drop a specific member (group must have other members)
ALTER DATABASE DROP LOGFILE MEMBER '/u01/oradata/ORCL/redo01a.log';

-- Note: you cannot drop the CURRENT group's members
-- Switch the log first if needed:
ALTER SYSTEM SWITCH LOGFILE;
```

### Drop a Log Group

```sql
-- Drop entire group (must not be CURRENT or ACTIVE)
ALTER DATABASE DROP LOGFILE GROUP 4;

-- Then delete the physical file from OS:
-- rm /u01/oradata/ORCL/redo04a.log
-- rm /u02/oradata/ORCL/redo04b.log
```

### Force a Log Switch

```sql
-- Force Oracle to switch to the next group (and archive the current)
ALTER SYSTEM SWITCH LOGFILE;

-- Force a checkpoint (flush dirty blocks to disk)
ALTER SYSTEM CHECKPOINT;
```

---

## Recommended Multiplexing Layout

```
Disk A          Disk B          Disk C (optional)
────────        ────────        ─────────────────
control01.ctl   control02.ctl   control03.ctl
redo01a.log     redo01b.log
redo02a.log     redo02b.log
redo03a.log     redo03b.log
datafiles...    (different path)
```

!!! danger "Never put control file copies on the same disk"
    If both copies of a control file are on the same disk and that disk fails, you lose both — defeating the purpose of multiplexing.
