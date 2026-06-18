# Oracle DBA2 Reference Site

Personal reference site for Oracle DBA2 (Workshop 2) — RMAN, Backup, Recovery. Built with MkDocs Material dark theme, auto-deployed to GitHub Pages.

## 🚀 Deploy in 5 Steps

### Step 1 — Create GitHub repo
1. Go to https://github.com/new
2. Name: `oracle-dba2-ref`
3. Visibility: **Public**
4. Do NOT initialize with README
5. Click **Create repository**

### Step 2 — Update site URL
Edit `mkdocs.yml` line 3:
```yaml
site_url: https://ahmedr0001.github.io/oracle-dba2-ref/
```

### Step 3 — Push to GitHub
```bash
git init
git add .
git commit -m "Oracle DBA2 reference site — Lectures 1 & 2"
git branch -M main
git remote add origin https://github.com/ahmedr0001/oracle-dba2-ref.git
git push -u origin main
```

### Step 4 — Enable GitHub Pages
1. Go to repo → **Settings** → **Pages**
2. Source: **Deploy from a branch**
3. Branch: **gh-pages** / folder: **/ (root)**
4. Click **Save**

> The `gh-pages` branch appears automatically ~2 minutes after your first push.

### Step 5 — Visit your site
```
https://ahmedr0001.github.io/oracle-dba2-ref/
```

---

## 🛠️ Local Preview
```bash
pip install -r requirements.txt
mkdocs serve
# → http://127.0.0.1:8000
```

## ✏️ Adding Future Lectures
Add new `.md` files under `docs/` and add them to the `nav:` section in `mkdocs.yml`. Push — GitHub Actions rebuilds automatically.

## 📁 Structure
```
docs/
├── index.md                          ← Home page
├── quick-reference.md                ← All commands, one page
├── 01-rman-intro/                    ← Lecture 1: RMAN Fundamentals
│   ├── index.md
│   ├── why-backup.md
│   ├── rpo-rto.md
│   ├── rman-architecture.md
│   ├── rman-connections.md
│   ├── rman-configuration.md
│   └── catalog-db.md
├── 02-backup-types/                  ← Lecture 1: Storage & Redo
│   ├── index.md
│   ├── fra.md
│   ├── multiplexing.md
│   └── archivelog.md
├── 03-backup-commands/               ← Lecture 2: Backup Commands
│   ├── index.md
│   ├── backup-types.md
│   ├── whole-partial.md
│   ├── controlfile.md
│   ├── archivelogs.md
│   ├── backup-as-copy.md
│   └── more-options.md
└── 04-backup-types/                  ← Lecture 2: Consistent vs Inconsistent
    └── consistent-inconsistent.md
```
