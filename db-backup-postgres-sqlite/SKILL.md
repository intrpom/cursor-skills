---
name: db-backup-postgres-sqlite
description: Implements a production database backup system for Next.js projects using PostgreSQL (production) and SQLite (development). Exports all tables to JSON files, validates record counts, rotates old backups, and optionally uploads to Dropbox. Includes a restore script that syncs production data to local SQLite for development. Use when setting up database backups, adding a backup script, implementing backup/restore workflow, or protecting production data before migrations. Based on a proven production pattern from ales-kalina.cz projects.
---

# DB Backup: PostgreSQL → JSON → SQLite

Production-tested backup system for Next.js projects with dual-database setup (PostgreSQL in prod, SQLite locally). Exports every table to individual JSON files, validates completeness, and supports optional Dropbox cloud upload.

Reference implementation: `/Users/aleskalina/CascadeProjects/skola-event/scripts/backup/`

## Stack

- `postgres` (PostgreSQL client) + `better-sqlite3` (local SQLite)
- `dropbox` (optional cloud upload)
- `dotenv` for env loading
- Plain Node.js ESM scripts — no framework dependencies

## File Structure

```
scripts/backup/
├── backup.sh                  # Entry point — loads env, runs backup, cleanup
├── backup-db.mjs              # Core: exports tables → JSON, validates
├── restore-to-local.mjs       # Restores latest backup into local SQLite
├── upload-to-dropbox.mjs      # Optional: uploads backup folder to Dropbox
└── verify-backup-coverage.mjs # Optional: checks all DB tables are in backup

backups/database/
├── backup-2025-11-21T12-00-00-000Z/
│   ├── users.json
│   ├── webinar-registrations.json
│   ├── metadata.json          # timestamp, table counts, notes
│   └── ...
└── backup-2025-11-20T.../ (max 5 kept)
```

---

## Setup

### 1. Install packages

```bash
npm install postgres better-sqlite3 dotenv
npm install dropbox          # only if using Dropbox upload
```

### 2. Environment variables

```bash
# .env.production or .env.production.local (Vercel pull)
POSTGRES_URL=postgres://...

# .env.local (for Dropbox, optional)
DROPBOX_ACCESS_TOKEN=sl.xxx
```

Pull from Vercel if needed:
```bash
npx vercel env pull .env.production.local
```

---

## Core Script: `backup-db.mjs`

Three key responsibilities:

### A. Sensitive column filtering

Tables with passwords or secrets need custom queries. All other tables use `SELECT *` automatically — no manual maintenance when new tables are added.

```js
const SENSITIVE_QUERIES = {
  users: `
    SELECT id, name, email, created_at, last_login
    FROM users ORDER BY id
  `,
  admin_users: `
    SELECT id, username, created_at
    FROM admin_users ORDER BY id
  `,
}

async function fetchTable(tableName) {
  const customQuery = SENSITIVE_QUERIES[tableName]
  if (customQuery) return await sql.unsafe(customQuery)
  return await sql`SELECT * FROM ${sql(tableName)}`
}
```

### B. Auto-discover all tables

```js
async function getAllTables() {
  const result = await sql`
    SELECT table_name FROM information_schema.tables
    WHERE table_schema = 'public' AND table_type = 'BASE TABLE'
      AND table_name NOT LIKE '_prisma%' AND table_name NOT LIKE 'pg_%'
    ORDER BY table_name
  `
  return result.map(t => t.table_name)
}
```

### C. Validation — compare backup counts vs live DB

After export, re-count every table and compare. Tolerates small race conditions (new rows during backup), fails hard if counts are significantly off.

```js
const CRITICAL_TABLES = ['users', 'registrations', 'payments']

// tolerance: log tables allow ±10, others ±5
// hasErrors = true → process.exit(1)
```

Each backup folder also gets `metadata.json`:
```json
{
  "timestamp": "2025-11-21T12:00:00.000Z",
  "database": "PostgreSQL (production)",
  "tables": 12,
  "totalRecords": 4821,
  "tableCounts": { "users": 312, "registrations": 4200, ... },
  "note": "Passwords not included: users, admin_users"
}
```

---

## Entry Point: `backup.sh`

Handles env loading (strips Vercel's quoted values), then delegates to Node scripts.

```bash
#!/bin/bash
set -e

# Load POSTGRES_URL — tries .env.production, then .env.production.local, then .env.local
if [ -f ".env.production.local" ]; then
  while IFS= read -r line; do
    [[ -n "$line" && ! "$line" =~ ^# ]] && \
      export "$(echo "$line" | sed 's/="\(.*\)"$/=\1/')"
  done < ".env.production.local"
fi

node scripts/backup/backup-db.mjs

# Cleanup: keep only 5 most recent backups
cd backups/database
ls -t | grep "backup-" | tail -n +6 | xargs -r rm -rf

# Optional: verify all DB tables are covered
node scripts/backup/verify-backup-coverage.mjs
```

Usage:
```bash
./scripts/backup/backup.sh              # basic backup
./scripts/backup/backup.sh --dropbox    # backup + upload to Dropbox
```

---

## Restore Script: `restore-to-local.mjs`

Syncs latest production backup into local SQLite. Handles schema drift between prod and dev:

- Auto-adds missing columns (infers type from data)
- Drops `NOT NULL` constraints if production has NULL values
- Removes `UNIQUE` constraints if production has duplicates
- Uses `INSERT OR REPLACE` to handle any remaining conflicts
- Skips `admin_users` (passwords not in backup — keep local admin accounts)

```bash
node scripts/backup/restore-to-local.mjs
```

Output:
```
📦 Restoring from: backup-2025-11-21T12-00-00-000Z
🗑️  Clearing existing data...
🔧 Checking schema...
   + registrations.stripe_customer_id (TEXT)
📥 Importing data...
   ✅  users: 312 records
   ✅  registrations: 4200 records
✅ Restore complete! 12 tables, 4821 records
```

---

## Dropbox Upload: `upload-to-dropbox.mjs`

```js
import { Dropbox } from 'dropbox'

export async function uploadToDropbox(backupDir) {
  const dbx = new Dropbox({ accessToken: process.env.DROPBOX_ACCESS_TOKEN })
  const folderName = path.basename(backupDir)   // e.g. backup-2025-11-21T...
  const dropboxPath = `/your-app-backups/${folderName}`

  for (const file of fs.readdirSync(backupDir)) {
    await dbx.filesUpload({
      path: `${dropboxPath}/${file}`,
      contents: fs.readFileSync(path.join(backupDir, file)),
      mode: { '.tag': 'overwrite' },
    })
  }
}
```

Env required: `DROPBOX_ACCESS_TOKEN` in `.env.local`

Get a long-lived token: Dropbox App Console → your app → OAuth2 → Generate access token

---

## Workflow Rules

**Before any DB modification** (migrations, seed scripts, schema changes):
```bash
./scripts/backup/backup.sh
node scripts/backup/restore-to-local.mjs
# Now work with local SQLite safely
```

**For read-only inspection** — use existing backup JSON files directly, no need to run backup first.

**Never create data directly in local SQLite** — always go through the admin panel or API in production, then restore.

---

## `.gitignore` & `.gitattributes`

```gitignore
# Keep backups in git (max 5, small JSON files)
backups/database/
!backups/database/backup-*/
```

Or exclude entirely if backups are large:
```gitignore
backups/
```

---

## Checklist

- [ ] `POSTGRES_URL` available (`.env.production` or `.env.production.local`)
- [ ] `backups/database/` directory created (or script creates it)
- [ ] `SENSITIVE_QUERIES` covers all tables with passwords/secrets
- [ ] `CRITICAL_TABLES` list updated for your project
- [ ] `backup.sh` is executable: `chmod +x scripts/backup/backup.sh`
- [ ] Test restore: `node scripts/backup/restore-to-local.mjs`
- [ ] Add `./scripts/backup/backup.sh` to pre-migration checklist
