---
tags: [automated-backup, cron, shell-script, incremental]
---

# Automated Backup

Production backups should never run manually. The standard pattern: write a shell script that calls RMAN with all commands inside it, then schedule it with cron.

---

## Step 1 — Create Script Directory

```bash
# As the oracle OS user
cd ~
mkdir -p scripts
```

---

## Step 2 — Write the RMAN Shell Script

```bash
vi /home/oracle/scripts/RMAN_backup.sh
```

```bash
#!/bin/bash
# /home/oracle/scripts/RMAN_backup.sh

# Oracle environment must be set — cron doesn't inherit your shell variables
ORACLE_SID=ORCL
ORACLE_HOME=/u01/app/oracle/product/19.0/dbhome_1
export ORACLE_SID ORACLE_HOME
PATH=$ORACLE_HOME/bin:$PATH
export PATH

# Log file with date in name so each run has its own log
LOGFILE=/home/oracle/scripts/RMAN_$(date +%Y%m%d_%H%M).log

# Call RMAN, pass all commands via heredoc, append output to log
$ORACLE_HOME/bin/rman log=$LOGFILE append << EOF
connect target '/';

set echo on;  -- print each command to the log before running it

run {
  -- One I/O worker channel (increase number for parallel backup)
  ALLOCATE CHANNEL ch1 DEVICE TYPE DISK;

  -- Level 1 incremental backup including archive logs, with a tag
  BACKUP INCREMENTAL LEVEL 1
    TAG 'incr_update'
    DATABASE PLUS ARCHIVELOG;

  -- Delete backups that exceed the retention policy from disk
  DELETE NOPROMPT OBSOLETE;

  -- Delete archive logs older than 1 day (already safely backed up above)
  DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-1';

  -- Free the channel resource when done
  RELEASE CHANNEL ch1;
}

exit;
EOF
```

---

## Step 3 — Make Executable

```bash
chmod 774 /home/oracle/scripts/RMAN_backup.sh
```

---

## Step 4 — Schedule with Cron

```bash
crontab -e
```

```bash
# Run daily at 06:00 AM
0 6 * * * /home/oracle/scripts/RMAN_backup.sh

# Run every 4 hours
0 0,4,8,12,16,20 * * * /home/oracle/scripts/RMAN_backup.sh
```

---

## Step 5 — Test It

```bash
# Run manually first to confirm it works
/home/oracle/scripts/RMAN_backup.sh

# Check the output log
cat /home/oracle/scripts/RMAN_$(date +%Y%m%d)*.log
```

---

## Smart Weekly Script (Level 0 Sunday, Level 1 Mon-Sat)

```bash
#!/bin/bash
ORACLE_SID=ORCL
ORACLE_HOME=/u01/app/oracle/product/19.0/dbhome_1
export ORACLE_SID ORACLE_HOME
PATH=$ORACLE_HOME/bin:$PATH; export PATH

LOGFILE=/home/oracle/scripts/RMAN_$(date +%Y%m%d_%H%M).log

# date +%u: 1=Mon ... 7=Sun
DOW=$(date +%u)
if [ "$DOW" -eq 7 ]; then LEVEL=0; else LEVEL=1; fi  # Level 0 on Sunday only

$ORACLE_HOME/bin/rman log=$LOGFILE append << EOF
connect target '/';
set echo on;
run {
  ALLOCATE CHANNEL ch1 DEVICE TYPE DISK;
  BACKUP AS COMPRESSED BACKUPSET
    INCREMENTAL LEVEL $LEVEL
    TAG 'weekly_strategy'
    DATABASE PLUS ARCHIVELOG;
  DELETE NOPROMPT OBSOLETE;
  DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-2';
  RELEASE CHANNEL ch1;
}
exit;
EOF
```

```bash
# One cron entry covers the whole week
# 0 2 * * * /home/oracle/scripts/RMAN_weekly.sh
```

---

## Section Commands Summary

```bash
mkdir -p /home/oracle/scripts          # create script directory
chmod 774 /home/oracle/scripts/RMAN_backup.sh  # make executable
/home/oracle/scripts/RMAN_backup.sh   # test manually
crontab -e                             # edit cron schedule
crontab -l                             # verify cron entries
```
