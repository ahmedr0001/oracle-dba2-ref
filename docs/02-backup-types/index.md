---
tags: [fra, multiplexing, archivelog]
---

# 💾 Storage & Redo

This section covers the physical storage infrastructure that makes backup and recovery possible.

```
Oracle Storage for Recoverability
──────────────────────────────────
Online Redo Logs  → capture every change as it happens
     │
     ▼ (when log fills, LGWR switches)
Archived Redo Logs → permanent history of all changes
     │
     ▼ (RMAN reads these during recovery)
Fast Recovery Area → central storage for all recovery files
```

| Topic | What You'll Learn |
|---|---|
| [Fast Recovery Area (FRA)](fra.md) | Configure FRA, what lives there, monitor space |
| [Multiplexing](multiplexing.md) | Protect control files and redo logs against single-point failure |
| [Archive Log Mode](archivelog.md) | Enable ARCHIVELOG, set destination, manage archived logs |
