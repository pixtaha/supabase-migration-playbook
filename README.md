# Supabase Database Migration Guide

A battle-tested, step-by-step guide to migrate a Supabase project (schema + data) between two free-tier accounts using `pg_dump` and `psql`. Covers everything from pre-flight checks to post-migration tasks, troubleshooting, and rollback.

> **Use case:** Moving a Supabase Postgres database from one account/email to another, while keeping the source project intact (so this is a **copy**, not a transfer).

---

## Table of Contents

1. [Overview](#overview)
2. [Pre-flight Checklist](#pre-flight-checklist)
3. [Environment Setup](#environment-setup)
4. [Migration Steps](#migration-steps)
5. [Verification](#verification)
6. [Post-Migration Tasks](#post-migration-tasks)
7. [Troubleshooting](#troubleshooting)
8. [Rollback Plan](#rollback-plan)
9. [Security Notes](#security-notes)
10. [Appendix: Useful Queries](#appendix-useful-queries)

---

## Overview

### What this guide does

- Dumps the **schema** (tables, views, indexes, triggers, foreign keys) from a source Supabase project.
- Dumps the **data** from all tables in the `public` schema.
- Restores both into a brand-new Supabase project on a different account.
- Configures the destination project (timezone, etc.).
- Verifies the migration with row-count comparisons.

### What this guide does NOT cover

- Migrating Supabase Auth users (`auth.users`).
- Migrating Storage buckets and files.
- Migrating Edge Functions.
- Migrating Realtime subscriptions config.
- Migrating Vault secrets.

If your project relies on any of the above, plan those separately.

### Estimated time

- **Small project** (under 1000 rows total): 15–20 minutes.
- **Medium project** (under 10,000 rows): 30–45 minutes.
- **First time doing it:** add 30 minutes for reading and troubleshooting.

---

## Pre-flight Checklist

Before you touch anything, confirm all of these:

| # | Item | Why |
|---|------|-----|
| 1 | You have **owner-level access** to both source and destination Supabase accounts | You need to dump from one and restore to the other |
| 2 | The **source project is healthy** and not paused | Free-tier projects pause after 7 days of inactivity |
| 3 | You know the **source database password** (or can reset it) | Required for `pg_dump` |
| 4 | You have a Mac/Linux/Windows machine with terminal access | All commands are CLI-based |
| 5 | You have at least **200MB free disk space** for backups | Dumps can be a few hundred MB depending on data |
| 6 | You've identified **which schemas to migrate** (usually just `public`) | This guide migrates `public` only |
| 7 | You understand that **the source project will not be modified** | This is a copy, not a transfer |
| 8 | You have a **secure place to store passwords** (password manager) | You'll need to handle DB passwords carefully |

If any of the above is unclear, stop and resolve it first.

---

## Environment Setup

### 1. Install PostgreSQL client tools

You need `pg_dump` and `psql` matching the Postgres version of Supabase (currently **PostgreSQL 17**). Using an older version will fail with `server version mismatch`.

#### macOS (with Homebrew)

```bash
brew install postgresql@17
echo 'export PATH="/opt/homebrew/opt/postgresql@17/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

#### Linux (Ubuntu/Debian)

```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt update
sudo apt install postgresql-client-17
```

#### Windows

Download the PostgreSQL 17 installer from [postgresql.org/download/windows](https://www.postgresql.org/download/windows/) and install only the **Command Line Tools** component. Add the install directory's `bin/` folder to your PATH.

### 2. Verify the installation

```bash
pg_dump --version
psql --version
```

Both should report `(PostgreSQL) 17.x` or higher. If you see version 16 or lower, fix the PATH or reinstall.

### 3. Create a working directory

Keep all migration artifacts (schema, data, scripts) in one folder you can back up later:

```bash
mkdir -p ~/supabase-migration/CLIENT_NAME
cd ~/supabase-migration/CLIENT_NAME
```

Replace `CLIENT_NAME` with something meaningful (e.g., `acme-gym-2026-05`).

---

## Migration Steps

### Step 1 — Get the source connection string

1. Log in to Supabase Dashboard with the **source account**.
2. Open the source project.
3. Click **Connect** at the top.
4. Choose the **Session pooler** tab. **Do not use Direct connection** — it requires IPv6, which many networks (and CI/CD platforms) don't support.
5. Note the values:
   - `host` — looks like `aws-X-REGION.pooler.supabase.com`
   - `port` — `5432`
   - `database` — `postgres`
   - `user` — `postgres.PROJECT_ID`
6. Have the **database password** ready (this is the Postgres password, not your account password). If you don't have it, reset it from **Project Settings → Database → Reset database password**.

> **Tip:** Set the password as an environment variable to avoid repeating it. Replace `YOUR_PASSWORD` with the actual password:
> ```bash
> export SOURCE_PW='YOUR_PASSWORD'
> ```
> Then in subsequent commands, use `PGPASSWORD="$SOURCE_PW"` instead of typing the password each time.

### Step 2 — Test the source connection

```bash
PGPASSWORD="$SOURCE_PW" psql \
  -h aws-X-REGION.pooler.supabase.com \
  -p 5432 \
  -U postgres.SOURCE_PROJECT_ID \
  -d postgres \
  -c "SELECT current_user, current_database(), version();"
```

Expected output: a single row showing the user, `postgres` as the database, and `PostgreSQL 17.x`. If you see an error, jump to [Troubleshooting](#troubleshooting).

### Step 3 — Dump the schema

```bash
PGPASSWORD="$SOURCE_PW" pg_dump \
  -h aws-X-REGION.pooler.supabase.com \
  -p 5432 \
  -U postgres.SOURCE_PROJECT_ID \
  -d postgres \
  --schema-only \
  --no-owner \
  --no-privileges \
  --schema=public \
  -f schema.sql
```

**Flag explanations:**

| Flag | Purpose |
|------|---------|
| `--schema-only` | Dump structure only, no data |
| `--no-owner` | Don't include ownership commands (which would fail on the destination because we're not superuser) |
| `--no-privileges` | Skip GRANT/REVOKE commands (Supabase manages these) |
| `--schema=public` | Only dump the `public` schema — leave Supabase-internal schemas (`auth`, `storage`, `realtime`, etc.) alone |

### Step 4 — Clean the schema dump

`pg_dump` adds a few statements that will fail on a fresh Supabase project. Remove them:

```bash
sed -E \
  -e '/^\\restrict/d' \
  -e '/^\\unrestrict/d' \
  -e '/^CREATE SCHEMA public;/d' \
  -e '/^COMMENT ON SCHEMA public/d' \
  schema.sql > schema_clean.sql
```

**Why these lines cause problems:**

- `\restrict` / `\unrestrict` — `psql` 17 meta-commands that don't exist in older versions and are unnecessary here.
- `CREATE SCHEMA public` — the `public` schema already exists on every Supabase project.
- `COMMENT ON SCHEMA public` — modifying schema comments requires permissions you don't have.

### Step 5 — Dump the data

```bash
PGPASSWORD="$SOURCE_PW" pg_dump \
  -h aws-X-REGION.pooler.supabase.com \
  -p 5432 \
  -U postgres.SOURCE_PROJECT_ID \
  -d postgres \
  --data-only \
  --no-owner \
  --no-privileges \
  --schema=public \
  -f data.sql
```

### Step 6 — Prepare the data dump for restore

`pg_dump` adds `ALTER TABLE ... DISABLE TRIGGER ALL` statements that **fail on Supabase** because you can't disable system triggers without superuser privileges. The clean way to bypass this is to wrap the dump with `session_replication_role = replica`, which Postgres allows:

```bash
{
  echo "SET session_replication_role = replica;"
  cat data.sql
  echo "SET session_replication_role = DEFAULT;"
} > data_final.sql
```

**What this does:**

- `session_replication_role = replica` tells Postgres to skip user-defined triggers and foreign key checks for the rest of this session.
- After the data is loaded, we reset it to `DEFAULT` so triggers fire normally afterward.
- This makes the import idempotent regardless of foreign key dependencies between rows.

> **Note:** This works for data-only restores. Do not use this trick for schema migrations.

### Step 7 — Create the destination project

1. Log in to Supabase with the **destination account**.
2. Create an organization if you don't have one (free tier).
3. **New Project** with these settings:
   - **Project Name:** anything descriptive
   - **Database Password:** click **Generate a password** and save it immediately in a password manager
   - **Region:** ideally the same region as the source for lower latency, but any region works fine
   - **Plan:** Free
   - **Security:** leave defaults (Enable Data API + Automatically expose new tables)
4. Click **Create new project** and wait 2–3 minutes for it to provision. The dashboard should show **Status: Healthy** before you proceed.

### Step 8 — Get the destination connection string

Same as Step 1, but on the destination account. Note the new `host`, `user`, and `password`.

Set the password:

```bash
export DEST_PW='NEW_PROJECT_PASSWORD'
```

### Step 9 — Test the destination connection

```bash
PGPASSWORD="$DEST_PW" psql \
  -h aws-Y-REGION.pooler.supabase.com \
  -p 5432 \
  -U postgres.DEST_PROJECT_ID \
  -d postgres \
  -c "SELECT current_user, current_database(), version();"
```

If this fails with `tenant/user not found`, wait another 2 minutes — the project is still initializing.

### Step 10 — Restore the schema

```bash
PGPASSWORD="$DEST_PW" psql \
  -h aws-Y-REGION.pooler.supabase.com \
  -p 5432 \
  -U postgres.DEST_PROJECT_ID \
  -d postgres \
  -v ON_ERROR_STOP=1 \
  -f schema_clean.sql
```

**Expected output:** a stream of `CREATE TABLE`, `CREATE VIEW`, `CREATE INDEX`, `ALTER TABLE`, `CREATE TRIGGER` messages. **No** `ERROR` lines.

The `-v ON_ERROR_STOP=1` flag halts on the first error, so you'll know immediately if something went wrong.

### Step 11 — Restore the data

```bash
PGPASSWORD="$DEST_PW" psql \
  -h aws-Y-REGION.pooler.supabase.com \
  -p 5432 \
  -U postgres.DEST_PROJECT_ID \
  -d postgres \
  -v ON_ERROR_STOP=1 \
  --single-transaction \
  -f data_final.sql
```

**Expected output:** a stream of `COPY N` lines (where N is the row count per table), followed by `setval` lines (auto-increment IDs reset), and a final `SET` line.

`--single-transaction` wraps the entire restore in one transaction — if anything fails, **nothing** is committed, so you can fix and retry without partial data.

---

## Verification

### Compare row counts between source and destination

Run this query against **both** databases and compare the results:

```sql
SELECT 
  table_name,
  (xpath('/row/c/text()', 
    query_to_xml(format('SELECT COUNT(*) AS c FROM public.%I', table_name), false, true, '')
  ))[1]::text::int AS row_count
FROM information_schema.tables
WHERE table_schema = 'public' 
  AND table_type = 'BASE TABLE'
ORDER BY table_name;
```

From the terminal:

```bash
# Source
PGPASSWORD="$SOURCE_PW" psql -h SOURCE_HOST -p 5432 -U SOURCE_USER -d postgres -c "<query above>"

# Destination
PGPASSWORD="$DEST_PW" psql -h DEST_HOST -p 5432 -U DEST_USER -d postgres -c "<query above>"
```

**Expected:** every row count matches exactly. If any differs, see [Troubleshooting](#troubleshooting).

### Verify views exist

```sql
SELECT table_name FROM information_schema.views WHERE table_schema = 'public';
```

The list should match the source.

### Spot-check a record

Pick one row from a critical table on the source and confirm it exists on the destination:

```sql
SELECT * FROM your_critical_table WHERE id = SOMETHING LIMIT 1;
```

---

## Post-Migration Tasks

These are easy to forget. Do them right after a successful migration.

### 1. Set the database timezone (if you need a non-UTC timezone)

By default, Supabase Postgres uses **UTC**. To make `NOW()` and `created_at` defaults return your local time, set the timezone on the database **and** on each role used by clients:

```bash
PGPASSWORD="$DEST_PW" psql -h DEST_HOST -p 5432 -U DEST_USER -d postgres -c "
ALTER DATABASE postgres SET timezone TO 'YOUR_TIMEZONE';
ALTER ROLE postgres SET timezone TO 'YOUR_TIMEZONE';
ALTER ROLE authenticator SET timezone TO 'YOUR_TIMEZONE';
ALTER ROLE anon SET timezone TO 'YOUR_TIMEZONE';
ALTER ROLE authenticated SET timezone TO 'YOUR_TIMEZONE';
ALTER ROLE service_role SET timezone TO 'YOUR_TIMEZONE';
"
```

Replace `YOUR_TIMEZONE` with a valid IANA timezone like `Asia/Dubai`, `Africa/Cairo`, `Europe/London`, etc.

> **Why both `DATABASE` and `ROLE`?** The Session Pooler reuses connections from a pool that may not pick up `ALTER DATABASE` defaults. Setting it on each role ensures new connections start with the right timezone regardless of how they connect.

**Verify with a fresh connection:**

```bash
PGPASSWORD="$DEST_PW" psql -h DEST_HOST -p 5432 -U DEST_USER -d postgres -c "SELECT NOW();"
```

The output should show the timezone offset (e.g., `+04` for Dubai).

> **About existing data:** Rows already migrated keep their `created_at` values as UTC literals. If your columns are `timestamp without time zone`, those values are now reinterpreted as local time — meaning a row created at `11:00 UTC` (which was 15:00 Dubai time) now displays as `11:00 Dubai`. If this matters, you'll need a one-time UPDATE to shift them. For new rows going forward, everything is correct.

### 2. Update API keys in client apps

The destination project has **completely different** API keys than the source. Update every client (n8n, mobile app, frontend, etc.) with the new values from **Project Settings → API Keys**:

- `anon` (public) key
- `service_role` (secret) key
- New project URL (`https://NEW_PROJECT_ID.supabase.co`)

### 3. Update database credentials in integrations

If you have automation tools (n8n, Make, Zapier, etc.) connecting directly to Postgres, update their connection settings to use the new host, user, and password.

### 4. Re-enable Row Level Security (RLS) if needed

If your source project had RLS policies, they were migrated as part of the schema. But if you were running with RLS disabled and want to enable it on the new project:

```sql
ALTER TABLE public.your_table ENABLE ROW LEVEL SECURITY;
```

Then write your policies as needed.

### 5. Sanity-check the application flow

Run through one or two end-to-end flows in your app to confirm:

- Reads work
- Writes work
- Triggers fire correctly
- Views return expected data
- Timezone-sensitive logic produces correct results

### 6. Archive the migration artifacts

Save the working directory (`schema.sql`, `schema_clean.sql`, `data.sql`, `data_final.sql`) to a safe location — cloud storage or a private GitHub repo. They are useful for:

- Future migrations of the same project
- Disaster recovery if the destination project is accidentally deleted
- Reference when troubleshooting later

**Never commit dumps with sensitive data to public repos.**

---

## Troubleshooting

### `pg_dump: error: server version mismatch`

**Cause:** Your `pg_dump` is older than the Supabase server version (currently Postgres 17).

**Fix:** Install Postgres 17 client tools and verify with `pg_dump --version`.

### `could not translate host name "db.PROJECT_ID.supabase.co" to address`

**Cause:** You're trying to use the **Direct connection** string, which requires IPv6. Your network probably doesn't support IPv6.

**Fix:** Use the **Session pooler** connection string instead. It works over IPv4 and looks like `aws-X-REGION.pooler.supabase.com`.

### `FATAL: password authentication failed for user "postgres"`

**Possible causes:**

1. **Wrong password.** Double-check or reset from Dashboard.
2. **You used `-U postgres` instead of `-U postgres.PROJECT_ID`.** The Session pooler requires the full user including the project reference.
3. **Special characters in the password** got mangled by your shell. Wrap the connection string in single quotes, or use `PGPASSWORD='...'` as a separate environment variable rather than embedding it in a URL.

### `FATAL: (ENOTFOUND) tenant/user postgres.XXXX not found`

**Cause:** The destination project is still initializing, or the user string has a typo.

**Fix:** Wait 2–3 minutes after project creation. Re-copy the user from the dashboard. Confirm the region matches what's in the dashboard.

### `psql:data.sql:N: ERROR: permission denied: "RI_ConstraintTrigger_..." is a system trigger`

**Cause:** The data dump includes `ALTER TABLE ... DISABLE TRIGGER ALL`, which requires superuser permissions you don't have on Supabase.

**Fix:** Use the `data_final.sql` approach in [Step 6](#step-6--prepare-the-data-dump-for-restore) to wrap the dump in `SET session_replication_role = replica`.

### `ERROR: relation "TABLE_NAME" does not exist` during data restore

**Cause:** A function or trigger references a table without specifying its schema (e.g., `chat_sessions` instead of `public.chat_sessions`). At restore time, `search_path` is empty, so the unqualified name can't be resolved.

**Fix:** This is a latent bug in the source schema. Two options:

1. **Workaround (recommended for the migration):** Wrap the data restore in `SET session_replication_role = replica` (see Step 6). This skips trigger execution during restore.
2. **Permanent fix:** Edit the function/trigger to use `public.TABLE_NAME` and re-apply via SQL Editor.

### Row counts don't match between source and destination

**Possible causes:**

1. **The restore was interrupted.** If you ran without `--single-transaction`, partial data may have been committed. Truncate the destination tables and re-run the restore.
2. **The source changed during the dump.** If writes happened to the source mid-migration, the destination will be slightly behind. Re-run the dump from the source.
3. **A foreign key violation aborted a `COPY`.** Check the restore output for errors; the `session_replication_role = replica` wrapper should prevent this.

### `SHOW timezone` returns `UTC` even after `ALTER DATABASE`

**Cause:** The Session Pooler caches connections and may not pick up new database defaults.

**Fix:** Apply the timezone setting to roles as well (see Post-Migration Task 1). Also try disconnecting and reconnecting to force a fresh session.

### Mystery extra tables in the destination

If you see tables you don't recognize, they were likely created during testing on the source. Decide whether to keep or drop them:

```sql
DROP TABLE IF EXISTS public.unwanted_table CASCADE;
```

> Be **very careful** with `CASCADE` — it will also drop anything that depends on the table.

---

## Rollback Plan

If the migration goes wrong, here's how to recover.

### Scenario 1: Restore failed partway through

Because we used `--single-transaction`, **nothing was committed**. The destination is in the same state as before the restore. You can:

1. Identify the cause from the error message.
2. Fix the issue (e.g., adjust the schema, edit the dump file).
3. Re-run the same restore command.

### Scenario 2: Restore "succeeded" but data is wrong/corrupt

Truncate everything and start over:

```sql
-- Run on the DESTINATION ONLY
DO $$
DECLARE
  r record;
BEGIN
  FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname = 'public') LOOP
    EXECUTE 'TRUNCATE TABLE public.' || quote_ident(r.tablename) || ' RESTART IDENTITY CASCADE;';
  END LOOP;
END $$;
```

Then re-run Steps 10–11.

### Scenario 3: You need to abandon the destination entirely

The source project was never modified, so it's untouched. You can:

1. Delete the broken destination project from the Supabase Dashboard.
2. Create a new destination project.
3. Re-run from Step 7.

### Scenario 4: You accidentally modified the source

This is the scenario you really want to avoid, which is why this guide only ever **reads** from the source. If you somehow modified the source:

- **Free tier:** there's no automatic backup. You may be able to ask Supabase support for help, but assume the modification is permanent.
- **Pro tier:** restore from the daily backup via Dashboard → Database → Backups.

> **Best practice:** Never run write queries against the source during migration. Treat it as read-only.

---

## Security Notes

### Password handling

- **Never** commit a database password to git, even in a private repo.
- **Never** paste a password in a chat, ticket, or shared document.
- Store passwords in a password manager (1Password, Bitwarden, etc.).
- Use environment variables (`PGPASSWORD`) when running commands, not inline in the URL.
- If a password is accidentally exposed, **reset it immediately** from the Supabase Dashboard.

### Connection string handling

The connection string includes the user and host but not the password. It's still sensitive — anyone with the password can connect. Treat it as a secret, but it's less critical than the password itself.

### Dump files

`schema.sql` is mostly safe to share for debugging — it only contains structure.

`data.sql` and `data_final.sql` contain **all your production data**, including potentially personal information (names, phone numbers, etc.). Treat them like a database backup:

- Store them encrypted if possible.
- Don't commit them to public repos.
- Delete local copies once you're sure the migration is verified.
- If your jurisdiction has data protection laws (GDPR, PDPL, etc.), the dump files are personal data — handle accordingly.

### Reset passwords after migration

After a successful migration, consider rotating both source and destination database passwords. This is especially important if the passwords were ever exposed during the process.

---

## Appendix: Useful Queries

### Count rows in all public tables

```sql
SELECT 
  table_name,
  (xpath('/row/c/text()', 
    query_to_xml(format('SELECT COUNT(*) AS c FROM public.%I', table_name), false, true, '')
  ))[1]::text::int AS row_count
FROM information_schema.tables
WHERE table_schema = 'public' 
  AND table_type = 'BASE TABLE'
ORDER BY table_name;
```

### List all tables, views, and their types

```sql
SELECT table_name, table_type 
FROM information_schema.tables 
WHERE table_schema = 'public'
ORDER BY table_type, table_name;
```

### List all foreign key relationships

```sql
SELECT
  tc.table_name AS from_table,
  kcu.column_name AS from_column,
  ccu.table_name AS to_table,
  ccu.column_name AS to_column
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
  ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu
  ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND tc.table_schema = 'public';
```

### List all indexes

```sql
SELECT 
  schemaname,
  tablename,
  indexname,
  indexdef
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY tablename, indexname;
```

### List all triggers

```sql
SELECT 
  event_object_table AS table_name,
  trigger_name,
  event_manipulation,
  action_timing
FROM information_schema.triggers
WHERE trigger_schema = 'public'
ORDER BY event_object_table, trigger_name;
```

### Reset all sequences to match max IDs

Sometimes after a manual data load, sequences fall out of sync with the data. This regenerates them:

```sql
DO $$
DECLARE
  r record;
BEGIN
  FOR r IN (
    SELECT 
      pg_get_serial_sequence(quote_ident(table_schema) || '.' || quote_ident(table_name), column_name) AS seq,
      table_name,
      column_name
    FROM information_schema.columns
    WHERE table_schema = 'public'
      AND column_default LIKE 'nextval%'
  ) LOOP
    IF r.seq IS NOT NULL THEN
      EXECUTE format(
        'SELECT setval(%L, COALESCE((SELECT MAX(%I) FROM public.%I), 1), true);',
        r.seq, r.column_name, r.table_name
      );
    END IF;
  END LOOP;
END $$;
```

---

## Quick Reference Cheat Sheet

```bash
# 1. Set passwords as env vars
export SOURCE_PW='source_password'
export DEST_PW='destination_password'

# 2. Dump schema
PGPASSWORD="$SOURCE_PW" pg_dump -h SOURCE_HOST -p 5432 -U SOURCE_USER -d postgres \
  --schema-only --no-owner --no-privileges --schema=public -f schema.sql

# 3. Clean schema
sed -E -e '/^\\restrict/d' -e '/^\\unrestrict/d' \
       -e '/^CREATE SCHEMA public;/d' -e '/^COMMENT ON SCHEMA public/d' \
       schema.sql > schema_clean.sql

# 4. Dump data
PGPASSWORD="$SOURCE_PW" pg_dump -h SOURCE_HOST -p 5432 -U SOURCE_USER -d postgres \
  --data-only --no-owner --no-privileges --schema=public -f data.sql

# 5. Wrap data with replica role
{ echo "SET session_replication_role = replica;"; cat data.sql; echo "SET session_replication_role = DEFAULT;"; } > data_final.sql

# 6. Restore schema
PGPASSWORD="$DEST_PW" psql -h DEST_HOST -p 5432 -U DEST_USER -d postgres \
  -v ON_ERROR_STOP=1 -f schema_clean.sql

# 7. Restore data
PGPASSWORD="$DEST_PW" psql -h DEST_HOST -p 5432 -U DEST_USER -d postgres \
  -v ON_ERROR_STOP=1 --single-transaction -f data_final.sql

# 8. Set timezone (optional)
PGPASSWORD="$DEST_PW" psql -h DEST_HOST -p 5432 -U DEST_USER -d postgres -c "
ALTER DATABASE postgres SET timezone TO 'Asia/Dubai';
ALTER ROLE postgres SET timezone TO 'Asia/Dubai';
ALTER ROLE authenticator SET timezone TO 'Asia/Dubai';
ALTER ROLE anon SET timezone TO 'Asia/Dubai';
ALTER ROLE authenticated SET timezone TO 'Asia/Dubai';
ALTER ROLE service_role SET timezone TO 'Asia/Dubai';"
```

---

## Credits & Version

- **Tested with:** Supabase (free tier) running PostgreSQL 17, May 2026.
- **Tools used:** `pg_dump 17`, `psql 17`, Supabase Dashboard.
- **Maintainer:** Update this README every time you discover a new edge case or Supabase changes behavior.
