---
tags: [reporting, list, report, crosscheck]
---

# 📊 Reporting & Maintenance

Before you can trust a recovery, you need to know what backups exist, which are still valid, and which have gone stale. RMAN gives you three tools: LIST, REPORT, and CROSSCHECK.

| Topic | What You'll Learn |
|---|---|
| [LIST Command](list-command.md) | Query backup history — sets, copies, archivelogs |
| [REPORT Command](report-command.md) | What needs backing up, what's obsolete, DB schema |
| [Crosscheck & Delete](crosscheck-delete.md) | Verify backups exist on disk, clean up stale records |
| [Block Corruption](block-corruption.md) | Detect and repair corrupted blocks without full restore |
| [Block Change Tracking](bct.md) | Speed up incremental backups with BCT |
| [Automated Backup](automated-backup.md) | Shell script + cron setup for hands-free backups |
| [Dynamic Views](dynamic-views.md) | V$ and RC_ views for monitoring backups |
