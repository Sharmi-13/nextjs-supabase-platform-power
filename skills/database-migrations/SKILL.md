---
name: "database-migrations"
description: "Author, apply, and verify Supabase PostgreSQL migrations safely. Use when adding tables, columns, indexes, constraints, or any schema change to a Supabase project."
license: "MIT"
metadata:
  author: "Link2Skill Team"
  version: "1.0.0"
---

# Database Migrations

## Overview

Supabase uses timestamped SQL migration files in `supabase/migrations/`. This skill covers how to author safe, idempotent migrations; how to verify them before applying; and the rules that prevent data loss or broken deployments. All patterns are derived from the Link2Skill migration history.

## Prerequisites Checklist

- [ ] Supabase CLI installed (`npm install -g supabase` or via Homebrew)
- [ ] `supabase/migrations/` directory exists in the project
- [ ] Local Supabase running (`supabase start`) for testing migrations before push
- [ ] Never modify a migration that has already been applied to any environment

## The Golden Rules

1. **Never modify an applied migration.** Once a migration file has been run against any environment (local, staging, production), it is immutable. To change schema, create a new migration.
2. **Every new table must have RLS enabled.** See the `row-level-security` skill.
3. **Migrations must be idempotent where possible.** Use `IF NOT EXISTS`, `IF EXISTS`, `CREATE OR REPLACE`, and `DROP ... IF EXISTS` guards.
4. **Never delete data in a migration without a documented rollback plan.** Prefer soft-deletes or status columns.

## Step-by-Step Guide

### 1. Name migration files correctly

Files must be timestamped to guarantee execution order. Use the format:

```
YYYYMMDDHHMMSS_description.sql
```

Example: `20261001043131_content_is_demo.sql`

Generate a timestamp in PowerShell:
```powershell
Get-Date -Format "yyyyMMddHHmmss"
```

Always use a descriptive suffix that states what changes, not just the table name:
- ✅ `20261005120000_add_is_demo_to_content_items.sql`
- ❌ `20261005120000_content_items.sql`

### 2. Structure a migration file

Every migration file should follow this structure:

```sql
-- Migration: <short description>
--
-- Feature: <feature name>
-- Task: <task reference if applicable>
-- Requirements: <requirement numbers if applicable>
--
-- Purpose:
--   Explain WHY this change is being made, not just what it does.
--   Document the security and data integrity implications.
--
-- RLS impact: <none | describe changes>
-- Rollback: <how to undo if needed>

-- ── Schema changes ─────────────────────────────────────────────────────────

ALTER TABLE public.content_items
  ADD COLUMN IF NOT EXISTS is_demo BOOLEAN NOT NULL DEFAULT false;

-- ── Indexes ────────────────────────────────────────────────────────────────

CREATE INDEX IF NOT EXISTS idx_content_items_is_demo
  ON public.content_items (is_demo)
  WHERE is_demo = true;

-- ── Back-fill / data changes ───────────────────────────────────────────────

-- Idempotent UPDATE — safe to run multiple times
UPDATE public.content_items
SET is_demo = true
WHERE title IN ('Known seed title 1', 'Known seed title 2')
  AND is_demo = false;
```

### 3. Common safe migration patterns

#### Add a nullable column
```sql
ALTER TABLE public.users
  ADD COLUMN IF NOT EXISTS display_name VARCHAR(100);
```

#### Add a NOT NULL column with a default
```sql
ALTER TABLE public.content_items
  ADD COLUMN IF NOT EXISTS view_count INTEGER NOT NULL DEFAULT 0;
```

#### Add a NOT NULL column without a default (multi-step, safe for large tables)
```sql
-- Step 1: add as nullable
ALTER TABLE public.posts
  ADD COLUMN IF NOT EXISTS language_tag VARCHAR(20);

-- Step 2: back-fill existing rows
UPDATE public.posts SET language_tag = 'English' WHERE language_tag IS NULL;

-- Step 3: add NOT NULL constraint
ALTER TABLE public.posts
  ALTER COLUMN language_tag SET NOT NULL;
```

#### Create a new table
```sql
CREATE TABLE IF NOT EXISTS public.saved_items (
    learner_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    content_id UUID NOT NULL REFERENCES public.content_items(id) ON DELETE CASCADE,
    saved_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (learner_id, content_id)
);

-- Always enable RLS immediately after creating a table
ALTER TABLE public.saved_items ENABLE ROW LEVEL SECURITY;
```

#### Create a PostgreSQL enum type
```sql
CREATE TYPE content_type AS ENUM ('Post', 'Video_Lesson');
```

Adding values to an existing enum (Postgres 9.1+):
```sql
ALTER TYPE content_status ADD VALUE IF NOT EXISTS 'Archived';
```

Note: Removing values from an enum is not supported without recreating the type.

#### Create a partial index
```sql
-- Only index unread notifications — keeps the index small
CREATE INDEX IF NOT EXISTS idx_notifications_user_unread
  ON public.notifications(user_id, created_at DESC)
  WHERE is_read = FALSE;
```

#### Add a foreign key
```sql
ALTER TABLE public.tutor_profiles
  ADD CONSTRAINT tutor_profiles_user_fk
  FOREIGN KEY (id) REFERENCES public.users(id) ON DELETE CASCADE;
```

### 4. Soft-delete pattern

Never hard-delete rows from tables that other parts of the system may reference. Use a `deleted_at` or `deactivated_at` timestamp instead.

```sql
-- Add soft-delete column
ALTER TABLE public.tutor_profiles
  ADD COLUMN IF NOT EXISTS deactivated_at TIMESTAMPTZ;

-- Application-layer soft delete
UPDATE public.tutor_profiles
SET deactivated_at = NOW()
WHERE id = $1;

-- Queries filter out deactivated profiles at the application layer
-- (or via a view if the pattern is used widely)
```

### 5. Verify before applying

Always test migrations locally before pushing:

```bash
# Reset local DB and re-run all migrations from scratch
supabase db reset

# Apply pending migrations only (without reset)
supabase db push --local

# Inspect the current schema
supabase db dump --local
```

After applying, verify the change:
```sql
-- Check column exists with correct type and default
SELECT column_name, data_type, column_default, is_nullable
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'content_items'
  AND column_name = 'is_demo';

-- Check index was created
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'content_items'
  AND indexname = 'idx_content_items_is_demo';

-- Check RLS is enabled
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public'
  AND tablename = 'content_items';
```

### 6. Handle the handle_new_user trigger pattern

The `public.users` table is populated exclusively by a `SECURITY DEFINER` trigger on `auth.users`. Never INSERT directly into `public.users` from application code or migrations (except for seed data using the service role).

If you need to change the trigger:
```sql
-- Always use CREATE OR REPLACE — never DROP + CREATE in separate statements
CREATE OR REPLACE FUNCTION public.handle_new_user()
  RETURNS TRIGGER
  LANGUAGE plpgsql
  SECURITY DEFINER
  SET search_path = public   -- prevent search-path injection
AS $$
BEGIN
  -- ... logic
  RETURN NEW;
END;
$$;

-- Recreate the trigger only if replacing the function changes its signature
DROP TRIGGER IF EXISTS on_auth_user_created ON auth.users;
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

## Common Workflows

### Workflow: Add a feature flag column to an existing table

```sql
-- Migration: 20261010120000_add_featured_flag_to_content_items.sql
ALTER TABLE public.content_items
  ADD COLUMN IF NOT EXISTS is_featured BOOLEAN NOT NULL DEFAULT false;

CREATE INDEX IF NOT EXISTS idx_content_items_featured
  ON public.content_items (is_featured)
  WHERE is_featured = true;
```

### Workflow: Rename a column safely

Renaming a column is a breaking change if any deployed code references the old name. Use a two-migration approach:

```sql
-- Migration 1: add new column and keep old one
ALTER TABLE public.users ADD COLUMN IF NOT EXISTS display_name VARCHAR(100);
UPDATE public.users SET display_name = old_name WHERE display_name IS NULL;

-- Deploy app code that reads display_name (with fallback to old_name)

-- Migration 2: remove old column only after all app code is updated
ALTER TABLE public.users DROP COLUMN IF EXISTS old_name;
```

## Troubleshooting

### Error: "relation already exists"
**Cause:** Running a migration without `IF NOT EXISTS` guards after a partial failure.
**Solution:** Add `IF NOT EXISTS` to all `CREATE TABLE`, `CREATE INDEX`, and `ADD COLUMN` statements.

### Error: "cannot add a NOT NULL column without a default"
**Cause:** Adding a `NOT NULL` column to a table with existing rows and no default value.
**Solution:** Follow the three-step pattern in section 3: add nullable → back-fill → add constraint.

### Error: "column already exists"
**Cause:** Migration ran twice, or same column added in two migrations.
**Solution:** Use `ADD COLUMN IF NOT EXISTS` consistently.

### Migration applied to prod but not local (or vice versa)
**Cause:** Manual schema changes made outside migrations.
**Solution:** Never use the Supabase dashboard SQL editor for schema changes in environments that share migration history. All changes must go through migration files.

### Error: "violates foreign key constraint" during back-fill
**Cause:** Trying to set a foreign key column to a value that doesn't exist in the referenced table.
**Solution:** Ensure referenced rows exist before the back-fill UPDATE, or use a subquery: `WHERE id IN (SELECT id FROM referenced_table)`.

## Best Practices

- One logical change per migration file. Don't combine unrelated changes.
- Always comment the WHY, not just the WHAT.
- Use `IF NOT EXISTS` / `IF EXISTS` guards so migrations are safe to re-run after a reset.
- Add `NOT NULL` constraints and indexes in the same migration as the column they cover.
- Test `supabase db reset` before every push to catch ordering issues.
- Never mix DDL (schema changes) and DML (data changes) that depend on each other in the same statement block without explicit transaction awareness.
- Keep back-fill UPDATEs idempotent — always include a `WHERE ... = false` or equivalent guard.
