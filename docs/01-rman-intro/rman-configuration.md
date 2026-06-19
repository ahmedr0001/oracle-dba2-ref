---
tags: [rman, configuration, configure]
---

# RMAN Configuration

RMAN configuration parameters are **persistent** — stored in the control file or catalog, survive restarts, and apply to every backup unless overridden inside a `RUN {}` block.

```sql
RMAN> SHOW ALL;               -- view all current settings
RMAN> SHOW RETENTION POLICY;  -- view one specific setting
```

---

## 1 · Retention Policy

Controls when a backup is considered **obsolete** (safe to delete).

```sql
-- Keep backups that cover the last 7 days of recovery
-- Oracle keeps as many backups as needed to satisfy the window
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;

-- Keep only the last 3 most recent backups of each datafile
RMAN> CONFIGURE RETENTION POLICY TO REDUNDANCY 3;

-- No obsolete backups — everything is always valid
RMAN> CONFIGURE RETENTION POLICY TO NONE;

-- Reset to default (REDUNDANCY 1)
RMAN> CONFIGURE RETENTION POLICY CLEAR;
```

```sql
-- See which backups are now obsolete
RMAN> REPORT OBSOLETE;

-- Delete obsolete backups (asks for confirmation)
RMAN> DELETE OBSOLETE;

-- Delete without confirmation prompt (use in scripts)
RMAN> DELETE NOPROMPT OBSOLETE;
```

!!! info "RECOVERY WINDOW vs REDUNDANCY"
    `RECOVERY WINDOW OF 7 DAYS` guarantees you can restore to any point in the past 7 days.
    `REDUNDANCY 3` keeps the last 3 backups — simpler, but no point-in-time guarantee.

---

## 2 · Backup Optimization

Skips files that haven't changed since the last backup — mainly useful for read-only tablespaces.

```sql
-- Skip unchanged read-only datafiles in future backups
RMAN> CONFIGURE BACKUP OPTIMIZATION ON;

-- Back up everything every time (default)
RMAN> CONFIGURE BACKUP OPTIMIZATION OFF;

RMAN> CONFIGURE BACKUP OPTIMIZATION CLEAR;  -- reset to default
```

---

## 3 · Default Device Type

Where backups are written by default.

```sql
-- Write to disk/filesystem (default)
RMAN> CONFIGURE DEFAULT DEVICE TYPE TO DISK;

-- Write to tape via media manager
RMAN> CONFIGURE DEFAULT DEVICE TYPE TO SBT;

RMAN> CONFIGURE DEFAULT DEVICE TYPE CLEAR;
```

---

## 4 · Control File Auto Backup

Automatically backs up the control file and SPFILE after every backup operation.

```sql
-- Enable auto backup (recommended for production)
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;

-- Disable auto backup
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP OFF;

-- Set where auto backup files are written (%F = unique CF filename)
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP FORMAT
      FOR DEVICE TYPE DISK
      TO '/u01/rman_bkp/cf_%F';

-- Reset location to default (FRA or configured channel)
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP FORMAT
      FOR DEVICE TYPE DISK CLEAR;
```

!!! warning "Special rule — datafile 1 always triggers CF auto backup"
    Backing up datafile 1 (SYSTEM tablespace) always creates a control file backup automatically, even if AUTOBACKUP is OFF.

---

## 5 · Channels

A channel is the I/O worker process. Configure auto channels to set the backup path for all jobs.

```sql
-- Set default path/format for all disk channel backups
-- %d=DB name, %T=YYYYMMDD, %U=unique ID
RMAN> CONFIGURE CHANNEL DEVICE TYPE DISK
      FORMAT '/u01/rman_bkp/%d_%T_%U';

-- Two auto channels writing to different disks (parallelism)
RMAN> CONFIGURE CHANNEL 1 DEVICE TYPE DISK FORMAT '/disk1/rman/%U';
RMAN> CONFIGURE CHANNEL 2 DEVICE TYPE DISK FORMAT '/disk2/rman/%U';

-- Reset channel config to default
RMAN> CONFIGURE CHANNEL DEVICE TYPE DISK CLEAR;
```

Manual channel (overrides auto channel for one job only):

```sql
RMAN> RUN {
  -- Allocate a channel for this job only
  ALLOCATE CHANNEL ch1 DEVICE TYPE DISK FORMAT '/u01/rman_bkp/%U';
  BACKUP DATABASE;
  RELEASE CHANNEL ch1;  -- free the channel when done
}
```

---

## 6 · Parallelism

How many channels run at the same time — controls backup speed.

```sql
-- Run 2 I/O streams in parallel (good for 2 physical disks)
RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM 2;

-- Reset to 1 (no parallelism, default)
RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM CLEAR;
```

!!! tip "One channel per physical disk"
    Setting parallelism higher than your number of physical disks causes I/O contention and slows the backup down.

---

## 7 · Max Backup Set Size (MAXSETSIZE)

Limits how large a single backup set file can grow.

```sql
-- Limit each backup piece to 4 GB (useful for tape media)
RMAN> CONFIGURE MAXSETSIZE TO 4G;
RMAN> CONFIGURE MAXSETSIZE TO 2048M;

-- Unlimited (default)
RMAN> CONFIGURE MAXSETSIZE TO UNLIMITED;
RMAN> CONFIGURE MAXSETSIZE CLEAR;
```

!!! warning "Must be >= your largest single datafile"
    A single datafile cannot be split across backup pieces. If MAXSETSIZE is smaller than your biggest datafile, RMAN will error.

---

## 8 · Encryption

Encrypts backup data — important for backups stored offsite or in the cloud.

```sql
-- Enable encryption for all backups
RMAN> CONFIGURE ENCRYPTION FOR DATABASE ON;

-- Disable encryption
RMAN> CONFIGURE ENCRYPTION FOR DATABASE OFF;

-- Encrypt one tablespace only
RMAN> CONFIGURE ENCRYPTION FOR TABLESPACE users ON;

-- Set encryption algorithm (AES256 = strongest, AES128 = default)
RMAN> CONFIGURE ENCRYPTION ALGORITHM 'AES256';

RMAN> CONFIGURE ENCRYPTION FOR DATABASE CLEAR;
```

Password-based encryption (set before running the backup):

```sql
-- Use a passphrase — must supply same passphrase at restore time
RMAN> SET ENCRYPTION ON IDENTIFIED BY 'MyBackupPass';
RMAN> BACKUP DATABASE;
```

---

## Format String Variables

Used in FORMAT, CONFIGURE CHANNEL, and CONTROLFILE AUTOBACKUP FORMAT:

| Variable | Expands To | Example |
|---|---|---|
| `%d` | Database name | `ORCL` |
| `%T` | Full date YYYYMMDD | `20260618` |
| `%s` | Backup set number | `42` |
| `%p` | Piece number within set | `1` |
| `%U` | Unique ID (auto-generated) | `3df5k2a1_1_1` |
| `%F` | Unique control file auto backup name | `c-1234567890-20260618-00` |
| `%e` | Archive log sequence number | `157` |

---

## Section Commands Summary

```sql
RMAN> SHOW ALL;                                              -- view all settings
RMAN> SHOW RETENTION POLICY;                                 -- view one setting

-- Retention
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
RMAN> CONFIGURE RETENTION POLICY TO REDUNDANCY 3;
RMAN> CONFIGURE RETENTION POLICY CLEAR;

-- Optimization
RMAN> CONFIGURE BACKUP OPTIMIZATION ON;
RMAN> CONFIGURE BACKUP OPTIMIZATION OFF;

-- Device
RMAN> CONFIGURE DEFAULT DEVICE TYPE TO DISK;
RMAN> CONFIGURE DEFAULT DEVICE TYPE TO SBT;

-- Control file auto backup
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP OFF;
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/path/cf_%F';

-- Channels
RMAN> CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT '/path/%d_%T_%U';
RMAN> CONFIGURE CHANNEL 1 DEVICE TYPE DISK FORMAT '/disk1/rman/%U';
RMAN> CONFIGURE CHANNEL DEVICE TYPE DISK CLEAR;

-- Parallelism
RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM 2;
RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM CLEAR;

-- Max set size
RMAN> CONFIGURE MAXSETSIZE TO 4G;
RMAN> CONFIGURE MAXSETSIZE TO UNLIMITED;

-- Encryption
RMAN> CONFIGURE ENCRYPTION FOR DATABASE ON;
RMAN> CONFIGURE ENCRYPTION FOR DATABASE OFF;
RMAN> CONFIGURE ENCRYPTION ALGORITHM 'AES256';

-- Reset all to factory defaults
RMAN> CONFIGURE ALL CLEAR;

-- Obsolete management
RMAN> REPORT OBSOLETE;
RMAN> DELETE NOPROMPT OBSOLETE;
```
