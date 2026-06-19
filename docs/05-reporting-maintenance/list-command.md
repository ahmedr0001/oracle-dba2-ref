---
tags: [list, reporting, backup-sets, copies]
---

# LIST Command

LIST queries the RMAN repository (control file or catalog) and shows what backups and copies exist. It does **not** touch the actual files on disk — it reads metadata only.

---

## List Backup Sets

```sql
-- Show all backup sets (full detail)
RMAN> LIST BACKUP;

-- Summary view: one line per backup set (faster to read)
RMAN> LIST BACKUP SUMMARY;

-- Show only backup sets that RMAN thinks are missing from disk
-- (status EXPIRED — file was deleted but record still exists)
RMAN> LIST EXPIRED BACKUP;

-- All backups of the whole database
RMAN> LIST BACKUP OF DATABASE;

-- Backups of a specific tablespace
RMAN> LIST BACKUP OF TABLESPACE users;

-- Backups of a specific datafile (by number)
RMAN> LIST BACKUP OF DATAFILE 2;

-- Backups of archived redo logs
RMAN> LIST BACKUP OF ARCHIVELOG ALL;

-- Backups of the control file
RMAN> LIST BACKUP OF CONTROLFILE;

-- Lookup a specific backup set by its key number
RMAN> LIST BACKUPSET 42;

-- Lookup a backup set by its tag label
RMAN> LIST BACKUPSET TAG 'monthly_full_db_backup';
```

---

## List Image Copies

```sql
-- All image copies
RMAN> LIST COPY;

-- Image copies of the whole database
RMAN> LIST COPY OF DATABASE;

-- Image copies of a specific datafile
RMAN> LIST COPY OF DATAFILE 2;

-- Image copies completed in a date range
RMAN> LIST COPY OF DATAFILE 2
      COMPLETED BETWEEN '20-May-2025' AND '30-May-2025';
```

---

## List Archive Logs

```sql
-- All archive log records in the repository
RMAN> LIST ARCHIVELOG ALL;

-- Archive logs for a specific sequence range
RMAN> LIST ARCHIVELOG FROM SEQUENCE 100 UNTIL SEQUENCE 150;
```

---

## List Failures (Data Recovery Advisor)

```sql
-- Show detected database failures (used with Data Recovery Advisor)
RMAN> LIST FAILURE;

-- Show failures in detail
RMAN> LIST FAILURE DETAIL;
```

---

## Section Commands Summary

```sql
RMAN> LIST BACKUP;                                          -- all backups, full detail
RMAN> LIST BACKUP SUMMARY;                                  -- all backups, one-line summary
RMAN> LIST EXPIRED BACKUP;                                  -- backups marked missing
RMAN> LIST BACKUP OF DATABASE;                              -- whole DB backups
RMAN> LIST BACKUP OF TABLESPACE users;                      -- specific tablespace
RMAN> LIST BACKUP OF DATAFILE 2;                            -- specific datafile
RMAN> LIST BACKUP OF ARCHIVELOG ALL;                        -- archive log backups
RMAN> LIST BACKUP OF CONTROLFILE;                           -- control file backups
RMAN> LIST BACKUPSET 42;                                    -- by key number
RMAN> LIST BACKUPSET TAG 'monthly_full_db_backup';          -- by tag
RMAN> LIST COPY;                                            -- all image copies
RMAN> LIST COPY OF DATAFILE 2 COMPLETED BETWEEN 'date' AND 'date';
RMAN> LIST ARCHIVELOG ALL;                                  -- all archive log records
RMAN> LIST FAILURE;                                         -- Data Recovery Advisor failures
```
