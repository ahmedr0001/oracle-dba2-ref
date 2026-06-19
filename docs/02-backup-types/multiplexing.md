---
tags: [multiplexing, control-file, redo-log]
---

# Multiplexing

Multiplexing = keeping multiple identical copies of a critical file on **different physical disks**. If one disk fails, Oracle keeps running using the copies on the other disks — no downtime, no data loss.

Two files must be multiplexed: **control files** and **redo log members**.

---

## Control File Multiplexing

The control file is Oracle's master index. Lose all copies → database cannot mount. Recommended: 3 copies on 3 different disks.

### Check Current Control Files

```sql
-- Show paths of all current control files
SHOW PARAMETER CONTROL_FILES;

-- Alternative view
SELECT NAME, STATUS FROM V$CONTROLFILE;
```

### Add a Control File Copy

Oracle has no `ADD CONTROLFILE` command. You update the parameter and copy the file at OS level:

```sql
-- Step 1: check current locations
SHOW PARAMETER CONTROL_FILES;

-- Step 2: clean shutdown (never use ABORT here)
SHUTDOWN IMMEDIATE;
```

```bash
# Step 3: copy the control file to new location(s) at OS level
cp /u01/oradata/ORCL/control01.ctl /u02/oradata/ORCL/control02.ctl
cp /u01/oradata/ORCL/control01.ctl /u03/oradata/ORCL/control03.ctl
```

```sql
-- Step 4: tell Oracle about all copies (SPFILE = persistent, survives restart)
ALTER SYSTEM SET CONTROL_FILES =
  '/u01/oradata/ORCL/control01.ctl',
  '/u02/oradata/ORCL/control02.ctl',
  '/u03/oradata/ORCL/control03.ctl'
  SCOPE = SPFILE;   -- cannot use BOTH here; requires restart

-- Step 5: restart so Oracle reads the new SPFILE parameter
STARTUP;

-- Step 6: verify all three copies are now listed
SELECT NAME FROM V$CONTROLFILE;
```

---

## Redo Log Multiplexing

Each **group** has one or more **members** (physical files). Oracle writes to all members of the current group simultaneously. If one member's disk fails, Oracle uses the surviving member — no data loss.

Recommended: 3 groups, 2 members each, members on separate disks.

### Check Current Redo Logs

```sql
-- View groups: which is current, size, how many members
SELECT GROUP#, SEQUENCE#, BYTES/1024/1024 AS SIZE_MB, MEMBERS, STATUS
FROM V$LOG
ORDER BY GROUP#;

-- View members: physical file paths per group
SELECT GROUP#, MEMBER, STATUS
FROM V$LOGFILE
ORDER BY GROUP#, MEMBER;
```

**V$LOG STATUS values:**

| Status | Meaning |
|---|---|
| `CURRENT` | Oracle is writing to this group right now |
| `ACTIVE` | Used but not yet archived — needed for crash recovery |
| `INACTIVE` | Archived — safe to reuse or drop |
| `UNUSED` | Brand new group, never written to |

### Add a Member to an Existing Group

```sql
-- Add a second member to Group 1 (now it has 2 copies on different disks)
ALTER DATABASE ADD LOGFILE MEMBER
  '/u02/oradata/ORCL/redo01b.log'
  TO GROUP 1;

-- Add members to multiple groups in one command
ALTER DATABASE ADD LOGFILE MEMBER
  '/u02/oradata/ORCL/redo01b.log' TO GROUP 1,
  '/u02/oradata/ORCL/redo02b.log' TO GROUP 2,
  '/u02/oradata/ORCL/redo03b.log' TO GROUP 3;
```

### Add a New Group (With Members)

```sql
-- Add a completely new group with 2 members from the start
ALTER DATABASE ADD LOGFILE GROUP 4
  ('/u01/oradata/ORCL/redo04a.log',
   '/u02/oradata/ORCL/redo04b.log')
  SIZE 200M;
```

### Drop a Log Member

```sql
-- Drop one member from a group (group must have other members remaining)
ALTER DATABASE DROP LOGFILE MEMBER '/u01/oradata/ORCL/redo01a.log';
```

### Drop a Log Group

```sql
-- Drop entire group — must not be CURRENT or ACTIVE
-- Force a log switch first if needed to get it to INACTIVE
ALTER SYSTEM SWITCH LOGFILE;

-- Then drop the group
ALTER DATABASE DROP LOGFILE GROUP 4;
```

```bash
# Then delete the physical files from disk (Oracle doesn't do this automatically)
rm /u01/oradata/ORCL/redo04a.log
rm /u02/oradata/ORCL/redo04b.log
```

### Force Log Switch and Checkpoint

```sql
-- Force Oracle to stop writing to the current log and move to the next
-- This also triggers archiving of the current log
ALTER SYSTEM SWITCH LOGFILE;

-- Force a checkpoint: flush all dirty blocks from memory to disk
ALTER SYSTEM CHECKPOINT;
```

---

## Section Commands Summary

```sql
-- Control file
SHOW PARAMETER CONTROL_FILES;
SELECT NAME, STATUS FROM V$CONTROLFILE;
SHUTDOWN IMMEDIATE;
-- (cp files at OS level)
ALTER SYSTEM SET CONTROL_FILES = '/path/cf1.ctl','/path/cf2.ctl' SCOPE=SPFILE;
STARTUP;

-- Redo log groups and members
SELECT GROUP#, SEQUENCE#, MEMBERS, STATUS FROM V$LOG;
SELECT GROUP#, MEMBER, STATUS FROM V$LOGFILE ORDER BY GROUP#;

-- Add member to existing group
ALTER DATABASE ADD LOGFILE MEMBER '/disk_b/redo01b.log' TO GROUP 1;

-- Add new group with members
ALTER DATABASE ADD LOGFILE GROUP 4
  ('/disk_a/redo04a.log', '/disk_b/redo04b.log') SIZE 200M;

-- Drop member (group must have others)
ALTER DATABASE DROP LOGFILE MEMBER '/path/redo01a.log';

-- Drop group (must be INACTIVE)
ALTER DATABASE DROP LOGFILE GROUP 4;

-- Force switch and checkpoint
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM CHECKPOINT;
```
