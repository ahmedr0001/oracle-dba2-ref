---
tags: [backup, archivelog, archived-logs]
---

# Backup Archived Redo Logs

Archive logs are the bridge between a backup and the present. When you restore last Sunday's backup and the crash happened Wednesday, archive logs carry every transaction from Sunday to Wednesday — without them, that work is gone forever.

---

## Method 1 — Standalone Archive Log Backup

### Back Up All Logs

```sql
-- Back up all archive logs currently on disk
RMAN> BACKUP ARCHIVELOG ALL;

-- Back up and delete from disk after (saves space, keeps record in RMAN)
RMAN> BACKUP ARCHIVELOG ALL DELETE ALL INPUT;

-- Back up and delete only the files just backed up in this command
RMAN> BACKUP ARCHIVELOG ALL DELETE INPUT;
```

---

### Filter by SCN

SCN (System Change Number) is Oracle's internal transaction counter. Every commit gets a unique SCN.

```sql
-- Back up logs from SCN 1000 onwards
RMAN> BACKUP ARCHIVELOG FROM SCN 1000;

-- Back up logs up to SCN 2000
RMAN> BACKUP ARCHIVELOG UNTIL SCN 2000;

-- Back up logs in a specific SCN range
RMAN> BACKUP ARCHIVELOG FROM SCN 1000 UNTIL SCN 2000;

-- Alternative syntax
RMAN> BACKUP ARCHIVELOG SCN BETWEEN 1000 AND 2000;
```

```sql
-- Find the current SCN
SELECT CURRENT_SCN FROM V$DATABASE;
```

---

### Filter by Sequence Number

```sql
-- Back up from sequence 100 onwards
RMAN> BACKUP ARCHIVELOG FROM SEQUENCE 100;

-- Back up a specific range
RMAN> BACKUP ARCHIVELOG FROM SEQUENCE 100 UNTIL SEQUENCE 200;

-- Alternative syntax
RMAN> BACKUP ARCHIVELOG SEQUENCE BETWEEN 100 AND 200;
```

```sql
-- Find sequence numbers
SELECT SEQUENCE#, NAME, FIRST_TIME, NEXT_TIME
FROM V$ARCHIVED_LOG
WHERE DEST_ID = 1
ORDER BY SEQUENCE# DESC;
```

---

### Filter by Time

```sql
-- Back up logs from the last 24 hours
RMAN> BACKUP ARCHIVELOG FROM TIME 'SYSDATE-1';

-- Back up logs up to a specific date
RMAN> BACKUP ARCHIVELOG UNTIL TIME "TO_DATE('18/06/2026','DD/MM/YYYY')";

-- Back up logs in a specific time window
RMAN> BACKUP ARCHIVELOG
      FROM  TIME "TO_DATE('17/06/2026 20:00','DD/MM/YYYY HH24:MI')"
      UNTIL TIME "TO_DATE('18/06/2026 06:00','DD/MM/YYYY HH24:MI')";
```

---

## Method 2 — PLUS ARCHIVELOG (Recommended)

Adds archive log backup automatically to any database backup command.

```sql
-- Whole database + archive logs (most common production command)
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;

-- Compressed + archive logs
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;

-- Tablespace + archive logs
RMAN> BACKUP TABLESPACE users PLUS ARCHIVELOG;

-- Incremental + archive logs
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;
```

---

## Only Back Up What Hasn't Been Backed Up Yet

```sql
-- Skip archive logs already backed up once
RMAN> BACKUP ARCHIVELOG NOT BACKED UP 1 TIMES;

-- Skip logs backed up at least twice (use when you want dual copies)
RMAN> BACKUP ARCHIVELOG NOT BACKED UP 2 TIMES;
```

---

## List and Validate Archive Log Backups

```sql
-- List all archive log backups in RMAN repository
RMAN> LIST BACKUP OF ARCHIVELOG ALL;

-- List backup of a specific range
RMAN> LIST BACKUP OF ARCHIVELOG FROM SEQUENCE 100 UNTIL SEQUENCE 120;

-- Cross-check: verify archive log files actually exist on disk
RMAN> CROSSCHECK ARCHIVELOG ALL;

-- Delete catalog records for archive logs that no longer exist on disk
RMAN> DELETE EXPIRED ARCHIVELOG ALL;
```

---

## Delete Archive Logs Safely

```sql
-- Delete all (use with caution)
RMAN> DELETE NOPROMPT ARCHIVELOG ALL;

-- Delete logs older than 7 days
RMAN> DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-7';

-- Delete only logs already backed up
RMAN> DELETE NOPROMPT ARCHIVELOG ALL BACKED UP 1 TIMES TO DISK;
```

!!! warning "Never use 'rm' on archive logs"
    Always delete through RMAN so it updates its repository.

---

## Section Commands Summary

```sql
-- Backup all
RMAN> BACKUP ARCHIVELOG ALL;
RMAN> BACKUP ARCHIVELOG ALL DELETE INPUT;
RMAN> BACKUP ARCHIVELOG ALL DELETE ALL INPUT;
RMAN> BACKUP ARCHIVELOG NOT BACKED UP 1 TIMES;

-- By SCN
RMAN> BACKUP ARCHIVELOG FROM SCN 1000;
RMAN> BACKUP ARCHIVELOG UNTIL SCN 2000;
RMAN> BACKUP ARCHIVELOG SCN BETWEEN 1000 AND 2000;

-- By sequence
RMAN> BACKUP ARCHIVELOG FROM SEQUENCE 100;
RMAN> BACKUP ARCHIVELOG FROM SEQUENCE 100 UNTIL SEQUENCE 200;
RMAN> BACKUP ARCHIVELOG SEQUENCE BETWEEN 100 AND 200;

-- By time
RMAN> BACKUP ARCHIVELOG FROM TIME 'SYSDATE-1';
RMAN> BACKUP ARCHIVELOG UNTIL TIME "TO_DATE('18/06/2026','DD/MM/YYYY')";

-- With DB backup
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
RMAN> BACKUP TABLESPACE users PLUS ARCHIVELOG;
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;

-- List / validate
RMAN> LIST BACKUP OF ARCHIVELOG ALL;
RMAN> CROSSCHECK ARCHIVELOG ALL;
RMAN> DELETE EXPIRED ARCHIVELOG ALL;

-- Delete
RMAN> DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-7';
RMAN> DELETE NOPROMPT ARCHIVELOG ALL BACKED UP 1 TIMES TO DISK;

-- Find current SCN and sequence numbers
SELECT CURRENT_SCN FROM V$DATABASE;
SELECT SEQUENCE#, NAME, FIRST_TIME FROM V$ARCHIVED_LOG WHERE DEST_ID=1 ORDER BY SEQUENCE# DESC;
```
