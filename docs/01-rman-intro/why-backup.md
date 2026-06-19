---
tags: [rman, backup, failures]
---

# Why Backup & Failure Types

RMAN (Recovery Manager) is Oracle's built-in tool for backup and recovery. Unlike a simple `cp` command, RMAN understands Oracle's internal structure — it knows which blocks are used, which are corrupted, and exactly what's needed to recover.

---

## Categories of Failure

| Failure Type | Example | How It's Fixed |
|---|---|---|
| **User Process** | App crash, `CTRL+C` on a query | Automatic — Oracle rolls back uncommitted work |
| **Instance** | Server power cut, OS crash | Automatic instance recovery on next `STARTUP` |
| **Media / Storage** | Bad sector corrupts a `.dbf` file | Restore from backup + apply archive logs |
| **Inaccessible Component** | Wrong file permissions | Fix OS permissions or restore file |
| **Physical Corruption** | Disk write error damages a block | RMAN block media recovery |
| **Logical Corruption** | Accidental `DELETE` or `DROP TABLE` | Flashback or point-in-time recovery |
| **Disaster** | Fire, flood, data center outage | Offsite backup / Oracle Data Guard |

---

## Oracle Solutions by Recovery Speed

| Speed | Solution | Use Case |
|---|---|---|
| Seconds | Oracle RAC / Data Guard | Infrastructure failure |
| Minutes | Data Guard switchover / Flashback | Human error |
| Hours | RMAN restore | Media failure |
| Days | Offsite tape restore | Disaster |

!!! warning "A backup you can't restore is not a backup"
    Always test your recovery process before you need it.

---

## Section Commands

```sql
-- Not applicable here — this is a concepts page.
-- See the next pages for actual RMAN commands.
```
