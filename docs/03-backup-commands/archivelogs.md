---
tags: [backup, archivelog, archived-logs]
---

# Backup Archived Redo Logs

## Why Back Up Archive Logs?

Archive logs are the **bridge between a backup and the present**. When you restore a datafile from last Sunday's backup and the database crashed today (Wednesday), you need every archive log from Sunday through Wednesday to bring the database forward to the moment of failure.

Without archive log backups:
- Your restore point = the age of your last backup
- All work done since that backup is **permanently lost**

---

## Method 1: BACKUP ARCHIVELOG (Standalone)

### Back Up All Available Archive Logs

```sql
-- Back up all archive logs currently on disk
RMAN> BACKUP ARCHIVELOG ALL;

-- Back up all archive logs AND delete them from disk after backup
-- (frees disk space, keeps them in the backup repository)
RMAN> BACKUP ARCHIVELOG ALL DELETE ALL INPUT;

-- Delete only archive logs that have already been backed up
RMAN> BACKUP ARCHIVELOG ALL DELETE INPUT;
```

!!! info "DELETE INPUT vs DELETE ALL INPUT"
    - `DELETE INPUT` — deletes only the specific archive log files that were just backed up in this command
    - `DELETE ALL INPUT` — deletes all archive log copies (all destinations) after backing up

---

### Filter by SCN (System Change Number)

SCN is Oracle's internal version counter — every committed transaction gets a unique SCN. Use SCN-based filters for precise recovery point control.

```sql
-- Archive logs from SCN 1000 onwards
RMAN> BACKUP ARCHIVELOG FROM SCN 1000;

-- Archive logs up to SCN 2000
RMAN> BACKUP ARCHIVELOG UNTIL SCN 2000;

-- Archive logs between two SCNs (inclusive)
RMAN> BACKUP ARCHIVELOG FROM SCN 1000 UNTIL SCN 2000;

-- Equivalent BETWEEN syntax
RMAN> BACKUP ARCHIVELOG SCN BETWEEN 1000 AND 2000;
```

**Find the current SCN:**
```sql
SELECT CURRENT_SCN FROM V$DATABASE;
```

---

### Filter by Sequence Number

Archive logs are numbered sequentially. Use sequence numbers when you know exactly which logs you need.

```sql
-- From sequence 100 onwards
RMAN> BACKUP ARCHIVELOG FROM SEQUENCE 100;

-- From sequence 100 to 200 (inclusive)
RMAN> BACKUP ARCHIVELOG FROM SEQUENCE 100 UNTIL SEQUENCE 200;

-- Between syntax
RMAN> BACKUP ARCHIVELOG SEQUENCE BETWEEN 100 AND 200;
```

**Find archive log sequence numbers:**
```sql
SELECT SEQUENCE#, NAME, FIRST_TIME, NEXT_TIME
FROM V$ARCHIVED_LOG
WHERE DEST_ID = 1
ORDER BY SEQUENCE# DESC;
```

---

### Filter by Time

Use time-based filters when you think in terms of "last night" or "before the incident":

```sql
-- Archive logs generated in the last 24 hours
RMAN> BACKUP ARCHIVELOG FROM TIME 'SYSDATE-1';

-- Archive logs up to a specific date/time
RMAN> BACKUP ARCHIVELOG UNTIL TIME "TO_DATE('18/06/2026','DD/MM/YYYY')";

-- Archive logs in a specific time window
RMAN> BACKUP ARCHIVELOG
      FROM  TIME "TO_DATE('17/06/2026 20:00','DD/MM/YYYY HH24:MI')"
      UNTIL TIME "TO_DATE('18/06/2026 06:00','DD/MM/YYYY HH24:MI')";
```

---

## Method 2: BACKUP … PLUS ARCHIVELOG

The `PLUS ARCHIVELOG` clause adds archive log backup to any backup command. This is the **recommended approach** for complete backups because it ensures datafiles and their corresponding archive logs are backed up together.

```sql
-- Whole database + all archive logs (most common production backup)
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;

-- Compressed + archive logs
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;

-- Tablespace + its archive logs
RMAN> BACKUP TABLESPACE users PLUS ARCHIVELOG;

-- Incremental + archive logs
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;
```

**Execution sequence of `BACKUP DATABASE PLUS ARCHIVELOG`:**

```
1. ALTER SYSTEM SWITCH LOGFILE        ← archive the current online log
2. BACKUP ARCHIVELOG ALL              ← back up all archive logs
3. BACKUP DATABASE                    ← back up datafiles
4. ALTER SYSTEM SWITCH LOGFILE        ← archive logs generated during backup
5. BACKUP ARCHIVELOG ALL              ← back up those new logs
```

This sequence guarantees that after restore, you have everything needed to bring the database to a consistent state.

---

## Backup Archive Logs Not Yet Backed Up

Avoid re-backing up the same archive logs in incremental/frequent schedules:

```sql
-- Only back up archive logs that haven't been backed up yet
RMAN> BACKUP ARCHIVELOG NOT BACKED UP 1 TIMES;

-- Not backed up more than once (if you want dual copies)
RMAN> BACKUP ARCHIVELOG NOT BACKED UP 2 TIMES;
```

---

## List and Validate Archive Log Backups

```sql
-- List all archive log backups in repository
RMAN> LIST BACKUP OF ARCHIVELOG ALL;

-- List backup of specific sequence range
RMAN> LIST BACKUP OF ARCHIVELOG
      FROM SEQUENCE 100 UNTIL SEQUENCE 120;

-- Cross-check repository against actual files on disk
RMAN> CROSSCHECK ARCHIVELOG ALL;

-- Mark missing archive logs as EXPIRED
-- (files referenced in catalog that don't exist on disk)
RMAN> DELETE EXPIRED ARCHIVELOG ALL;
```

---

## Delete Archive Logs Safely

```sql
-- Delete all archive logs (use cautiously — check retention first)
RMAN> DELETE NOPROMPT ARCHIVELOG ALL;

-- Delete archive logs older than N days
RMAN> DELETE NOPROMPT ARCHIVELOG ALL
      COMPLETED BEFORE 'SYSDATE-7';

-- Delete only archive logs that are already backed up
RMAN> DELETE NOPROMPT ARCHIVELOG ALL
      BACKED UP 1 TIMES TO DISK;
```
