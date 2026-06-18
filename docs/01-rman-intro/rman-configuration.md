---
tags: [rman, configuration, configure]
---

# RMAN Configuration

RMAN has persistent configuration parameters stored in the control file (or catalog). They survive restarts and apply to every backup unless overridden in a `RUN` block.

```bash
# View all current settings
RMAN> SHOW ALL;

# View a specific setting
RMAN> SHOW RETENTION POLICY;
RMAN> SHOW DEFAULT DEVICE TYPE;
```

---

## 1 · Retention Policy

> **"How long is a backup considered valid before RMAN marks it obsolete?"**

This is the most critical configuration. It drives what `DELETE OBSOLETE` removes.

```sql
-- Option A: Recovery Window — backup is valid if it can recover the DB
-- to any point within the last N days
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;

-- Option B: Redundancy — keep the last N full backups of each datafile
-- Default is REDUNDANCY 1
RMAN> CONFIGURE RETENTION POLICY TO REDUNDANCY 3;

-- Option C: No retention — all backups are always valid (never obsolete)
RMAN> CONFIGURE RETENTION POLICY TO NONE;

-- Reset back to default (REDUNDANCY 1)
RMAN> CONFIGURE RETENTION POLICY CLEAR;
```

**Check and clean obsolete backups:**
```sql
-- List backups that are now obsolete
RMAN> REPORT OBSOLETE;

-- Delete obsolete backups (prompts for confirmation)
RMAN> DELETE OBSOLETE;

-- Delete without confirmation prompt (for scripts)
RMAN> DELETE NOPROMPT OBSOLETE;
```

!!! info "RECOVERY WINDOW vs REDUNDANCY"
    - `RECOVERY WINDOW OF 7 DAYS` — guarantees you can restore to any point in the past 7 days. Oracle keeps as many backups as needed to satisfy that window.
    - `REDUNDANCY 3` — always keeps the 3 most recent full backups. Simpler, but doesn't guarantee a specific point-in-time window.

---

## 2 · Backup Optimization

> **"Skip backing up files that haven't changed since the last backup."**

Most useful for **read-only tablespaces** that are backed up once and never need to be backed up again.

```sql
-- Enable optimization (skips unchanged read-only datafiles)
RMAN> CONFIGURE BACKUP OPTIMIZATION ON;

-- Disable (default — back up everything every time)
RMAN> CONFIGURE BACKUP OPTIMIZATION OFF;

RMAN> CONFIGURE BACKUP OPTIMIZATION CLEAR;  -- reset to default
```

**Use case:** You have a large 200 GB historical data tablespace set to READ ONLY. With optimization ON, RMAN skips it after the first backup, saving hours every night.

---

## 3 · Default Device Type

> **"Where does RMAN write backups by default?"**

```sql
-- Disk (default) — write to filesystem or ASM
RMAN> CONFIGURE DEFAULT DEVICE TYPE TO DISK;

-- SBT (System Backup to Tape) — write to tape library via media manager
RMAN> CONFIGURE DEFAULT DEVICE TYPE TO SBT;

RMAN> CONFIGURE DEFAULT DEVICE TYPE CLEAR;
```

| Device | Use When |
|---|---|
| `DISK` | Local filesystem, NFS mount, ASM — fast, easy |
| `SBT` | Tape libraries, Oracle Secure Backup (OSB), cloud via media agent |

---

## 4 · Control File Auto Backup

> **"Automatically back up the control file and SPFILE after every backup or structural change."**

```sql
-- Enable (recommended for production)
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;

-- Disable
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP OFF;

-- Set a specific location for the auto backup
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK
      TO '/u01/rman_bkp/cf_%F';
      -- %F = unique identifier (DB ID + timestamp)

RMAN> CONFIGURE CONTROLFILE AUTOBACKUP CLEAR;
```

!!! warning "Special rule for SYSTEM tablespace"
    Even if `CONTROLFILE AUTOBACKUP` is **OFF**, backing up **datafile 1** (the SYSTEM tablespace) will always trigger an automatic control file backup. This is hardcoded Oracle behavior.

---

## 5 · Channels

A channel is the I/O worker process. Configuring channels controls **where** backups are written and with what format.

### Auto Channel (Persistent)
Applies to every backup automatically:

```sql
-- Set format for all disk backups
RMAN> CONFIGURE CHANNEL DEVICE TYPE DISK
      FORMAT '/u01/rman_bkp/%d_%T_%U';
      -- %d = DB name, %T = YYYYMMDD, %U = unique ID

-- Multiple auto channels (parallelism)
RMAN> CONFIGURE CHANNEL 1 DEVICE TYPE DISK
      FORMAT '/disk1/rman/%U';
RMAN> CONFIGURE CHANNEL 2 DEVICE TYPE DISK
      FORMAT '/disk2/rman/%U';

RMAN> CONFIGURE CHANNEL DEVICE TYPE DISK CLEAR;
```

### Manual Channel (Per-Job)
Override channels for a specific backup inside a `RUN` block:

```sql
RMAN> RUN {
  ALLOCATE CHANNEL ch1 DEVICE TYPE DISK
    FORMAT '/u01/rman_bkp/%U';
  BACKUP DATABASE;
  RELEASE CHANNEL ch1;
}
```

!!! info "Manual channels override auto channels"
    When you manually allocate a channel inside `RUN {}`, RMAN ignores the configured auto channels for that job.

---

## 6 · Parallelism

> **"How many channels (I/O streams) run simultaneously?"**

```sql
-- Set degree of parallelism (= number of concurrent channels)
RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM 2;
RMAN> CONFIGURE DEVICE TYPE SBT  PARALLELISM 4;

RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM CLEAR;  -- reset to 1
```

**Rules:**
- Parallelism degree should equal the number of **physical disks** in your storage
- **Do not** set parallelism > 1 on a single-disk system — it causes contention and slows backups down
- Each degree of parallelism = one channel = one server process on the DB

---

## 7 · Backup Set Size (MAXSETSIZE)

> **"Limit how large a single backup set can grow."**

```sql
-- Limit backup set to 4 GB
RMAN> CONFIGURE MAXSETSIZE TO 4G;

-- In megabytes
RMAN> CONFIGURE MAXSETSIZE TO 2048M;

-- Reset to unlimited (default)
RMAN> CONFIGURE MAXSETSIZE TO UNLIMITED;
RMAN> CONFIGURE MAXSETSIZE CLEAR;
```

**Critical rules:**
- A single datafile **cannot be split** across backup sets — it must fit entirely in one backup piece
- `MAXSETSIZE` must be **≥ the size of the largest datafile** you're backing up
- If MAXSETSIZE is too small for the largest datafile, RMAN will error

**Use case:** Tape media has a fixed capacity. Set MAXSETSIZE to match the tape size so each tape holds exactly one backup set.

---

## 8 · Encrypted Backups

> **"Encrypt backup data so it's unreadable without the key — important for offsite/cloud backups."**

```sql
-- Enable encryption for all backups
RMAN> CONFIGURE ENCRYPTION FOR DATABASE ON;

-- Disable encryption
RMAN> CONFIGURE ENCRYPTION FOR DATABASE OFF;

-- Encrypt only a specific tablespace
RMAN> CONFIGURE ENCRYPTION FOR TABLESPACE users ON;

-- Set encryption algorithm (default AES128)
RMAN> CONFIGURE ENCRYPTION ALGORITHM 'AES256';

RMAN> CONFIGURE ENCRYPTION FOR DATABASE CLEAR;
```

!!! warning "Set a wallet/passphrase before encrypting"
    RMAN encryption requires an Oracle wallet or a passphrase. Without it, backups fail silently or cannot be restored.
    ```sql
    -- Transparent mode (uses wallet — must be open at restore time too)
    RMAN> SET ENCRYPTION ON;

    -- Password mode (uses a passphrase — specify at restore time)
    RMAN> SET ENCRYPTION ON IDENTIFIED BY 'MyBackupPass';
    ```

---

## Format String Substitution Variables

Used in `FORMAT`, `CONFIGURE CHANNEL`, and `CONFIGURE CONTROLFILE AUTOBACKUP FORMAT`:

| Variable | Expands To | Example |
|---|---|---|
| `%d` | Database name | `ORCL` |
| `%D` | Day of month (DD) | `18` |
| `%M` | Month (MM) | `06` |
| `%Y` | Year (YYYY) | `2026` |
| `%T` | Full date (YYYYMMDD) | `20260618` |
| `%s` | Backup set number | `42` |
| `%t` | Backup set timestamp (seconds since epoch) | `1718640000` |
| `%p` | Backup piece number within set | `1` |
| `%U` | Unique ID — `%u_%p_%c` combined | `3df5k2a1_1_1` |
| `%F` | Unique name for control file auto backup | `c-1234567890-20260618-00` |
| `%e` | Archived log sequence number | `157` |

**Recommended format string:**
```sql
RMAN> CONFIGURE CHANNEL DEVICE TYPE DISK
      FORMAT '/u01/rman/%d_%T_%s_%p_%U';
-- produces: ORCL_20260618_42_1_3df5k2a1_1_1
```

---

## Full Configuration Reset

```sql
-- Reset ALL RMAN configuration to factory defaults
RMAN> CONFIGURE ALL CLEAR;
```
