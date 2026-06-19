---
tags: [crosscheck, delete, expired, obsolete]
---

# Crosscheck & Delete

## Why Crosscheck Exists

RMAN's repository stores records of where backup files are supposed to be. If someone deletes a backup file with `rm` (bypassing RMAN), the repository still thinks it exists. CROSSCHECK reconciles the repository against what's actually on disk or tape — it marks missing files as **EXPIRED**.

---

## CROSSCHECK — Verify Files Exist

```sql
-- Check all backup sets against disk/tape
RMAN> CROSSCHECK BACKUP;

-- Check only backup sets (same as above but explicit)
RMAN> CROSSCHECK BACKUPSET;

-- Check all archive log copies
RMAN> CROSSCHECK ARCHIVELOG ALL;

-- Check image copies
RMAN> CROSSCHECK COPY;

-- Cross-check a specific backup set by key
RMAN> CROSSCHECK BACKUPSET 42;
```

After crosscheck, files that don't exist on disk are marked **EXPIRED** in the repository. Files that do exist remain **AVAILABLE**.

---

## DELETE EXPIRED — Remove Stale Repository Records

After a crosscheck, use DELETE EXPIRED to remove repository records that point to missing files. This cleans up the metadata — it does **not** try to delete anything from disk (the files are already gone).

```sql
-- Remove repository records for expired (missing) backup sets
RMAN> DELETE EXPIRED BACKUP;

-- Remove records for expired backup sets specifically
RMAN> DELETE EXPIRED BACKUPSET;

-- Remove records for expired archive log copies
RMAN> DELETE EXPIRED ARCHIVELOG ALL;

-- Remove records for expired image copies
RMAN> DELETE EXPIRED COPY;

-- No confirmation prompt (for scripts)
RMAN> DELETE NOPROMPT EXPIRED BACKUP;
```

---

## DELETE OBSOLETE — Remove Old But Valid Backups

OBSOLETE means the backup still exists on disk but is no longer needed per the retention policy (too old, or more recent backups make it redundant). Unlike EXPIRED, these files are actually deleted from disk.

```sql
-- Delete backups that exceed the configured retention policy
RMAN> DELETE OBSOLETE;

-- Delete without confirmation prompt
RMAN> DELETE NOPROMPT OBSOLETE;

-- Delete obsolete under a specific policy (override current config)
RMAN> DELETE OBSOLETE REDUNDANCY 2;
RMAN> DELETE OBSOLETE RECOVERY WINDOW OF 7 DAYS;
```

---

## DELETE Archive Logs

```sql
-- Delete all archive logs from disk (use carefully)
RMAN> DELETE NOPROMPT ARCHIVELOG ALL;

-- Delete archive logs older than 7 days
RMAN> DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-7';

-- Delete archive logs already backed up (safest approach)
RMAN> DELETE NOPROMPT ARCHIVELOG ALL BACKED UP 1 TIMES TO DISK;
```

---

## Typical Maintenance Workflow

```sql
-- Step 1: check which backup files still exist on disk
RMAN> CROSSCHECK BACKUP;
RMAN> CROSSCHECK ARCHIVELOG ALL;

-- Step 2: remove stale repository records for missing files
RMAN> DELETE NOPROMPT EXPIRED BACKUP;
RMAN> DELETE EXPIRED ARCHIVELOG ALL;

-- Step 3: delete old backups that are no longer needed
RMAN> DELETE NOPROMPT OBSOLETE;

-- Step 4: confirm what's left
RMAN> LIST BACKUP SUMMARY;
RMAN> REPORT NEED BACKUP;
```

---

## Section Commands Summary

```sql
-- Crosscheck
RMAN> CROSSCHECK BACKUP;
RMAN> CROSSCHECK BACKUPSET;
RMAN> CROSSCHECK ARCHIVELOG ALL;
RMAN> CROSSCHECK COPY;
RMAN> CROSSCHECK BACKUPSET 42;

-- Delete expired (stale repo records — files already gone)
RMAN> DELETE EXPIRED BACKUP;
RMAN> DELETE EXPIRED BACKUPSET;
RMAN> DELETE EXPIRED ARCHIVELOG ALL;
RMAN> DELETE EXPIRED COPY;
RMAN> DELETE NOPROMPT EXPIRED BACKUP;

-- Delete obsolete (old backups still on disk — removes them)
RMAN> DELETE OBSOLETE;
RMAN> DELETE NOPROMPT OBSOLETE;
RMAN> DELETE OBSOLETE REDUNDANCY 2;
RMAN> DELETE OBSOLETE RECOVERY WINDOW OF 7 DAYS;

-- Delete archive logs
RMAN> DELETE NOPROMPT ARCHIVELOG ALL;
RMAN> DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-7';
RMAN> DELETE NOPROMPT ARCHIVELOG ALL BACKED UP 1 TIMES TO DISK;
```
