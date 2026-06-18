---
tags: [backup, rman]
---

# 📦 Backup Commands

This section covers every RMAN BACKUP command — what to back up, how to back it up, and the options that control how backup sets are created.

```
BACKUP Command Structure:
─────────────────────────────────────────────────────────
BACKUP
  [AS BACKUPSET | AS COPY]          ← output type
  [COMPRESSED]                      ← save space
  [DEVICE TYPE DISK | SBT]          ← where to write
  [FORMAT 'path/%U']                ← naming pattern
  [TAG 'my_tag']                    ← label this backup
  [NOT BACKED UP [n TIMES]]         ← skip if already backed up
  target_spec                       ← WHAT to back up
  [PLUS ARCHIVELOG]                 ← also back up archive logs
  [DELETE [ALL] INPUT];             ← delete source after backup
```

| Topic | What You'll Learn |
|---|---|
| [Backup Types Explained](backup-types.md) | Full vs Incremental, Differential vs Cumulative |
| [Whole & Partial Backup](whole-partial.md) | DATABASE, TABLESPACE, DATAFILE commands |
| [Backup Control Files](controlfile.md) | Auto backup, manual backup, trace backup |
| [Backup Archived Logs](archivelogs.md) | By time, SCN, sequence, PLUS ARCHIVELOG |
| [Backup As Copy](backup-as-copy.md) | Image copies — instant recovery option |
| [More Backup Options](more-options.md) | Tags, NOT BACKED UP, device type, format strings |
