---
name: "row-level-security"
description: "Enable and write PostgreSQL Row Level Security policies for Supabase tables. Use when creating a new table, adding access controls, or auditing which users can read or modify rows."
license: "MIT"
metadata:
  author: "Link2Skill Team"
  version: "1.0.0"
---

# Row Level Security (RLS)

## Overview

Row Level Security is the primary data access control layer in a Supabase application. Every table must have RLS enabled with explicit policies. Without RLS, any authenticated user can read or modify any row. Policies are written in SQL using `auth.uid()` to identify the caller. All patterns here are drawn directly from the Link2Skill migration set.

## Prerequisites Checklist

- [ ] Table has been created in a prior migration
- [ ] You know which operations (SELECT, INSERT, UPDATE, DELETE) each user type needs
- [ ] You understand the ownership model for the table (who owns each row)
- [ ] RLS is never bypassed in application code — only SECURITY DEFINER functions may bypass it, and only when explicitly required

## The RLS Default Posture

**Default deny.** When RLS is enabled on a table and no policy matches, the operation is rejected. Never rely on the absence of a policy to block access — always explicitly enable RLS and write only the policies you intend.

```sql
-- Step 1: enable RLS (required for every new table)
ALTER TABLE public.my_table ENABLE ROW LEVEL SECURITY;

-- Step 2: add only the policies that are needed
-- If no INSERT policy exists → INSERT is denied for all users
-- If no DELETE policy exists → DELETE is denied for all users
```

## Step-by-Step Guide

### 1. Enable RLS in the same migration that creates the table

```sql
CREATE TABLE IF NOT EXISTS public.comments (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_id UUID NOT NULL REFERENCES public.content_items(id) ON DELETE CASCADE,
    author_id  UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    body       TEXT NOT NULL,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Always enable immediately — never leave a table without RLS
ALTER TABLE public.comments ENABLE ROW LEVEL SECURITY;
```

### 2. The four policy operations

Each policy covers one operation. Specify only the operations users actually need.

| Operation | Keyword | When used |
|-----------|---------|-----------|
| Read rows | `FOR SELECT` | Viewing data |
| Create rows | `FOR INSERT` | Writing new data |
| Modify rows | `FOR UPDATE` | Editing existing data |
| Remove rows | `FOR DELETE` | Hard-deleting rows |

For soft-delete patterns, omit `FOR DELETE` entirely and use an `UPDATE` to set `deleted_at`.

### 3. Policy anatomy

```sql
CREATE POLICY "descriptive_policy_name"
  ON public.table_name
  FOR SELECT          -- operation
  USING (             -- row-level filter: applied to existing rows
    auth.uid() = owner_id
  );
```

For INSERT and UPDATE, use both `USING` and `WITH CHECK`:

```sql
CREATE POLICY "table_insert_own"
  ON public.table_name
  FOR INSERT
  WITH CHECK (auth.uid() = owner_id);  -- validates the new/updated row

CREATE POLICY "table_update_own"
  ON public.table_name
  FOR UPDATE
  USING  (auth.uid() = owner_id)   -- which rows can be targeted
  WITH CHECK (auth.uid() = owner_id);  -- what the updated row must satisfy
```

The difference:
- `USING` — filters which rows the operation can target (read or modify).
- `WITH CHECK` — validates the row *after* the change (prevents changing ownership).

Always include both for UPDATE to prevent a user from changing `owner_id` to someone else's ID.

### 4. Real policy patterns from Link2Skill

#### Own-row access (users table)
```sql
ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;

-- Users can read their own row only
CREATE POLICY "users_select_own"
  ON public.users
  FOR SELECT
  USING (auth.uid() = id);

-- Users can update their own row only
CREATE POLICY "users_update_own"
  ON public.users
  FOR UPDATE
  USING (auth.uid() = id)
  WITH CHECK (auth.uid() = id);

-- INSERT omitted: rows created exclusively by handle_new_user trigger
-- DELETE omitted: handled by ON DELETE CASCADE from auth.users
```

#### Public read, owner write (tutor_profiles)
```sql
ALTER TABLE public.tutor_profiles ENABLE ROW LEVEL SECURITY;

-- Any authenticated user can read public profiles;
-- owners can always read their own profile regardless of visibility
CREATE POLICY "tutor_profiles_select_public"
  ON public.tutor_profiles
  FOR SELECT
  USING (
    auth.uid() IS NOT NULL
    AND (
      visibility = 'public'
      OR auth.uid() = id
    )
  );

CREATE POLICY "tutor_profiles_insert_own"
  ON public.tutor_profiles
  FOR INSERT
  WITH CHECK (auth.uid() = id);

CREATE POLICY "tutor_profiles_update_own"
  ON public.tutor_profiles
  FOR UPDATE
  USING  (auth.uid() = id)
  WITH CHECK (auth.uid() = id);

-- DELETE omitted: soft-delete via deactivated_at
```

#### Status-gated read, owner write (content_items)
```sql
ALTER TABLE public.content_items ENABLE ROW LEVEL SECURITY;

-- Authenticated users can read Published/Approved content;
-- owners can read their own content in any status (draft management)
CREATE POLICY "content_items_select"
  ON public.content_items
  FOR SELECT
  USING (
    auth.uid() IS NOT NULL
    AND (
      status IN ('Published', 'Approved')
      OR tutor_id = auth.uid()
    )
  );

CREATE POLICY "content_items_insert_own"
  ON public.content_items
  FOR INSERT
  WITH CHECK (auth.uid() = tutor_id);

CREATE POLICY "content_items_update_own"
  ON public.content_items
  FOR UPDATE
  USING  (auth.uid() = tutor_id)
  WITH CHECK (auth.uid() = tutor_id);

-- DELETE omitted: soft-delete via status = 'Deleted'
```

#### Notification-style: user can read/update own, no INSERT from client
```sql
ALTER TABLE public.notifications ENABLE ROW LEVEL SECURITY;

CREATE POLICY "notifications_select_own"
  ON public.notifications
  FOR SELECT
  USING (auth.uid() = user_id);

-- Users can mark their own notifications as read (UPDATE only)
CREATE POLICY "notifications_update_own"
  ON public.notifications
  FOR UPDATE
  USING  (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- INSERT omitted: notifications created server-side only
```

#### Junction table with follow/unfollow (follows)
```sql
ALTER TABLE public.follows ENABLE ROW LEVEL SECURITY;

-- Any authenticated user can read follow relationships
CREATE POLICY "follows_select"
  ON public.follows
  FOR SELECT
  USING (auth.uid() IS NOT NULL);

-- Learners can follow (insert their own rows)
CREATE POLICY "follows_insert_own"
  ON public.follows
  FOR INSERT
  WITH CHECK (auth.uid() = learner_id);

-- Learners can unfollow (delete their own rows)
CREATE POLICY "follows_delete_own"
  ON public.follows
  FOR DELETE
  USING (auth.uid() = learner_id);
```

### 5. Verify RLS is in effect

After applying a migration, always verify:

```sql
-- Check RLS is enabled on the table
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public' AND tablename = 'my_table';
-- rowsecurity must be true

-- List all policies on a table
SELECT policyname, cmd, qual, with_check
FROM pg_policies
WHERE schemaname = 'public' AND tablename = 'my_table';

-- Manually test as a specific user (requires service role in psql)
SET ROLE authenticated;
SET request.jwt.claims = '{"sub": "USER-UUID-HERE"}';
SELECT * FROM public.my_table;
RESET ROLE;
```

### 6. Omitting operations intentionally

Always document omitted operations in migration comments so future maintainers know it was deliberate:

```sql
-- INSERT intentionally omitted: rows created by server-side trigger only
-- DELETE intentionally omitted: soft-delete via deleted_at column
-- UPDATE intentionally omitted: records are immutable once created
```

## Common Workflows

### Workflow: Audit all tables for missing RLS

```sql
-- Find tables that do NOT have RLS enabled
SELECT tablename
FROM pg_tables
WHERE schemaname = 'public'
  AND tablename NOT IN (
    SELECT tablename FROM pg_tables
    WHERE schemaname = 'public' AND rowsecurity = true
  );
```

### Workflow: Check for tables with RLS enabled but no policies

```sql
-- Tables with RLS on but zero policies = effectively locked out for everyone
SELECT t.tablename
FROM pg_tables t
WHERE t.schemaname = 'public'
  AND t.rowsecurity = true
  AND NOT EXISTS (
    SELECT 1 FROM pg_policies p
    WHERE p.schemaname = 'public' AND p.tablename = t.tablename
  );
```

## Troubleshooting

### Error: "new row violates row-level security policy"
**Cause:** An INSERT or UPDATE was attempted where no matching policy allows it, or the `WITH CHECK` condition failed.
**Solution:** Confirm the correct user UUID is set in `auth.uid()` and that the table has a policy covering the operation. Check that the row's ownership column matches `auth.uid()`.

### SELECT returns 0 rows unexpectedly
**Cause:** RLS is filtering out all rows because no policy matches. Common when a new table is created with RLS enabled but policies haven't been added yet.
**Solution:** List policies with `SELECT * FROM pg_policies WHERE tablename = 'your_table'` and confirm the expected policy exists.

### Service role bypasses RLS — unexpected data visible in dashboard
**Cause:** The Supabase dashboard uses the service role key, which bypasses RLS entirely.
**Solution:** This is expected and correct. Always test your application using the anon key + user JWT, not the service role. Use the Supabase Table Editor's "RLS disabled" warning as a reminder.

### UPDATE policy allows changing the owner column
**Cause:** `WITH CHECK` is missing or doesn't re-check ownership after the update.
**Solution:** Always include `WITH CHECK (auth.uid() = owner_column)` on UPDATE policies, not just `USING`.

## Best Practices

- Enable RLS in the same migration that creates the table — never as a follow-up.
- Write the minimum set of policies needed. Omitting an operation is a deliberate denial.
- Always include both `USING` and `WITH CHECK` on UPDATE policies.
- Document intentionally omitted operations in migration comments.
- Test policies by querying as specific users, not as the service role.
- Never use `FORCE ROW LEVEL SECURITY` unless you have a specific need to also restrict the table owner.
- Prefer `auth.uid() IS NOT NULL` over checking session presence manually — it's the canonical Supabase idiom.
