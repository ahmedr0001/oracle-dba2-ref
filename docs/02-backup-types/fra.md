---
tags: [fra, fast-recovery-area, storage]
---

# Fast Recovery Area (FRA)

## What Is the FRA?

The Fast Recovery Area is a **single, unified storage location** (directory or ASM disk group) where Oracle automatically manages all files related to backup and recovery. Instead of scattering recovery files across multiple locations, the FRA gives Oracle one place to look.

```
FRA Location
─────────────────────────────────────
│ Control file copies                │  ← Permanent (used by the instance)
│ Online redo log members            │  ← Permanent
│                                   │
│ Archived redo logs                 │  ← Transient (can be deleted when backed up)
│ Flashback logs                     │  ← Transient
│ RMAN backup sets                   │  ← Transient
│ RMAN image copies                  │  ← Transient
│ Control file auto backups          │  ← Transient
└───────────────────────────────────┘
```

**Permanent** = files the instance actively uses — Oracle won't delete these automatically.
**Transient** = backup/recovery files — Oracle manages space by deleting old ones when needed.

---

## Configure the FRA

Two parameters, set together:

```sql
-- Set the FRA directory path (filesystem or ASM)
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST
  = '/u01/app/oracle/fast_recovery_area'
  SCOPE = BOTH;

-- Set the FRA size limit — Oracle will not exceed this
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST_SIZE = 20G
  SCOPE = BOTH;

-- Check current settings
SHOW PARAMETER DB_RECOVERY_FILE_DEST;
```

!!! warning "Set the SIZE before the DEST"
    Oracle validates that the size is sufficient when you set the destination. Set `DB_RECOVERY_FILE_DEST_SIZE` first to avoid errors.

---

## Disable the FRA

```sql
-- Remove the FRA destination (disables FRA)
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST = '' SCOPE = BOTH;
```

---

## What Happens When FRA Fills Up

Oracle uses a space management algorithm for the FRA:

1. First deletes **obsolete** backup sets (already beyond retention policy)
2. Then deletes **redundant** copies of files
3. Then deletes **already-backed-up** archive logs
4. If it still can't free space → **ORA-00257: archiver error**

```
ORA-00257: archiver error. Connect internal only, until freed.
```

This error means the archiver process cannot write new archive logs — **the database effectively stops accepting new transactions.**

### Fix ORA-00257

```bash
# Step 1: Connect as SYSDBA (even with ORA-00257 active)
sqlplus / as sysdba

# Step 2: Check FRA usage
SELECT * FROM V$RECOVERY_AREA_USAGE;

# Step 3: Option A — increase FRA size
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST_SIZE = 30G SCOPE=BOTH;

# Step 4: Option B — delete obsolete backups via RMAN
rman target /
RMAN> DELETE NOPROMPT OBSOLETE;
RMAN> DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-3';
```

---

## Monitor FRA Space

```sql
-- Summary: total space, used, reclaimable
SELECT * FROM V$RECOVERY_FILE_DEST;
-- Columns: NAME, SPACE_LIMIT, SPACE_USED, SPACE_RECLAIMABLE, NUMBER_OF_FILES

-- Breakdown by file type: what's consuming FRA space?
SELECT FILE_TYPE,
       PERCENT_SPACE_USED    AS PCT_USED,
       PERCENT_SPACE_RECLAIMABLE AS PCT_RECLAIMABLE,
       NUMBER_OF_FILES
FROM V$RECOVERY_AREA_USAGE;
```

**V$RECOVERY_AREA_USAGE FILE_TYPE values:**

| FILE_TYPE | What It Is |
|---|---|
| `CONTROL FILE` | Control file copies |
| `REDO LOG` | Online redo log members in FRA |
| `ARCHIVED LOG` | Archived redo log files |
| `BACKUP PIECE` | RMAN backup set pieces |
| `IMAGE COPY` | RMAN image copies |
| `FLASHBACK LOG` | Flashback Database logs |
| `FOREIGN ARCHIVED LOG` | Logs from standby/logical standby |

---

## FRA Best Practices

| Practice | Reason |
|---|---|
| Set FRA ≥ 3× the DB size | Room for backup sets + archive logs + flashback logs |
| Use a separate filesystem/disk for FRA | Avoid competing I/O with DB datafiles |
| Monitor `V$RECOVERY_AREA_USAGE` daily | Catch space issues before ORA-00257 hits |
| Enable CONTROLFILE AUTOBACKUP | Control file is automatically placed in FRA |
| Keep archive logs in FRA | Oracle auto-manages them when space is needed |
