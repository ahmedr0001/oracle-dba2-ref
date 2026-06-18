---
tags: [rman]
---

# ⚙️ RMAN Fundamentals

RMAN (Recovery Manager) is Oracle's built-in utility for backup, restore, and recovery. Unlike user-managed backups (OS-level copy), RMAN is block-aware — it knows what's used, what's corrupt, and what needs recovery.

| Topic | What You'll Learn |
|---|---|
| [Why Backup & Failure Types](why-backup.md) | Categories of failure, DBA responsibility |
| [RPO & RTO](rpo-rto.md) | Recovery objectives, downtime factors, protection tiers |
| [RMAN Architecture](rman-architecture.md) | Target DB, Catalog DB, channels, backup sets vs image copies |
| [RMAN Connections](rman-connections.md) | Connect syntax, log files, script files |
| [RMAN Configuration](rman-configuration.md) | All 8 CONFIGURE parameters with examples |
| [Catalog Database Setup](catalog-db.md) | Step-by-step: create CatDB, register target, resync |
