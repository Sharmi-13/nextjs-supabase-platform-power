---
name: "demo-and-seed-data"
description: "Safely create and manage demo and seed data for a Supabase learning platform. Use when seeding content for development, adding a shared demo flag to existing tables, or making seed data visible across multiple user accounts."
license: "MIT"
metadata:
  author: "Link2Skill Team"
  version: "1.0.0"
---

# Demo and Seed Data

## Overview

Seed data bootstraps the development environment with realistic content. In a multi-user platform, seed content created under one tutor's UUID is invisible to other tutors because queries filter by `tutor_id = auth.uid()`. The `is_demo` flag pattern solves this cleanly — without changing RLS policies or duplicating rows — by letting application queries opt into shared demo rows alongside private content.

## Prerequisites Checklist

- [ ] `content_items` table exists with RLS enabled
- [ ] A service role key is available for seeding (never the anon key)
- [ ] Seed scripts are in version control and reviewed before running
- [ ] Seed data uses realistic but clearly fake values (no real personal data)
- [ ] All seed titles are tracked in the migration back-fill list

## The is_demo Flag Pattern

### The problem

Demo content is seeded under one real tutor UUID. Any other tutor logging in sees an empty dashboard because `WHERE tutor_id = auth.uid()` excludes the demo rows.

### The solution

Add an `is_demo BOOLEAN NOT NULL DEFAULT false` column. Application queries include demo rows by ORing the flag:

```sql
-- Application query pattern
SELECT * FROM content_items
WHERE (tutor_id = auth.uid() OR is_demo = true)
  AND status IN ('Published', 'Approved');
```

This:
- Requires no RLS changes (the existing `content_items_select` policy already permits any authenticated user to read Published/Approved content).
- Keeps a single copy of each seed row.
- Is trivially removable in production (set `is_demo = false` on all rows, or filter `is_demo = false` at the query layer).

## Step-by-Step Guide

### 1. Add the is_demo column via migration

```sql
-- migration: YYYYMMDDHHMMSS_add_is_demo_to_content_items.sql

ALTER TABLE public.content_items
  ADD COLUMN IF NOT EXISTS is_demo BOOLEAN NOT NULL DEFAULT false;

-- Partial index: only index the demo rows — keeps the index tiny
CREATE INDEX IF NOT EXISTS idx_content_items_is_demo
  ON public.content_items (is_demo)
  WHERE is_demo = true;

-- Back-fill: mark rows that were seeded before this migration
-- Idempotent: the WHERE is_demo = false guard prevents double-setting
UPDATE public.content_items
SET is_demo = true
WHERE title IN (
  'Known seed title 1',
  'Known seed title 2'
)
AND is_demo = false;
```

### 2. Write the seed SQL file

Seed files live in `supabase/seed.sql` (applied by `supabase db reset`) or in a dedicated `supabase/seeds/` directory for manual seeding.

```sql
-- supabase/seed.sql
-- Demo seed data for development.
-- All rows are seeded under a fixed demo tutor UUID.
-- The is_demo flag makes them visible to all authenticated users.
--
-- NEVER run against production.

DO $$
DECLARE
  v_tutor_id UUID := '00000000-0000-0000-0000-000000000001';
BEGIN

-- Seed the demo tutor user (bypasses trigger — direct insert for seeding only)
INSERT INTO auth.users (id, email, encrypted_password, email_confirmed_at, raw_user_meta_data)
VALUES (
  v_tutor_id,
  'demo-tutor@example.com',
  crypt('DemoPassword123!', gen_salt('bf')),
  NOW(),
  '{"username": "demo_tutor", "roles": ["tutor"]}'::jsonb
)
ON CONFLICT (id) DO NOTHING;

-- The handle_new_user trigger creates the public.users row automatically.
-- Wait a moment or ensure trigger runs before inserting content.

-- Seed demo content
INSERT INTO public.content_items
  (id, tutor_id, type, title, body, subject_tag, language_tag, skill_level, status, is_demo)
VALUES
  (gen_random_uuid(), v_tutor_id, 'Post', 'Introduction to Algebra',
   'Algebra is the branch of mathematics...', 'Mathematics', 'English', 'Beginner', 'Published', true),

  (gen_random_uuid(), v_tutor_id, 'Post', 'கணிதம் — அடிப்படைகள்',
   'கணிதத்தின் அடிப்படை கோட்பாடுகள்...', 'Mathematics', 'Tamil', 'Beginner', 'Published', true),

  (gen_random_uuid(), v_tutor_id, 'Video_Lesson', 'Python Crash Course',
   'A complete introduction to Python programming.', 'Programming', 'English', 'Beginner', 'Published', true)

ON CONFLICT DO NOTHING;

END $$;
```

### 3. Query pattern including demo data

Whenever a tutor dashboard query fetches content, include demo rows:

```typescript
// lib/content/queries.ts
export async function getTutorContent(
  supabase: SupabaseClient,
  tutorId: string,
) {
  return supabase
    .from("content_items")
    .select("*")
    .or(`tutor_id.eq.${tutorId},is_demo.eq.true`)
    .in("status", ["Published", "Approved", "Draft", "Scheduled"])
    .order("created_at", { ascending: false });
}
```

For read-only feeds (learner discovery), demo rows appear naturally because `status IN ('Published', 'Approved')` already covers them via the existing RLS policy.

### 4. Reset and reseed locally

```bash
# Full reset: drops and recreates the local DB, runs all migrations, then seed.sql
supabase db reset

# Apply only pending migrations (no reset, no seed)
supabase db push --local
```

### 5. Identifying seed vs real data

Use meaningful, recognizable seed titles and a fixed UUID prefix for demo users. Never use real personal information in seed data.

```sql
-- Demo user IDs use a recognizable prefix
v_demo_tutor  UUID := '00000000-0000-0000-0000-000000000001';
v_demo_learner UUID := '00000000-0000-0000-0000-000000000002';
```

### 6. Preventing seed data from reaching production

Add a guard at the top of seed files:

```sql
-- Safety check: abort if this is not a local/dev environment
DO $$
BEGIN
  IF current_database() NOT LIKE '%local%'
     AND current_database() NOT LIKE '%dev%'
     AND current_database() NOT LIKE '%test%' THEN
    RAISE EXCEPTION
      'Seed script must not be run against a production database. '
      'Current database: %', current_database();
  END IF;
END $$;
```

Or, enforce at the CI/CD pipeline level: seed scripts are never in the production deployment pipeline.

## Common Workflows

### Workflow: Add new demo content without a migration

If the `is_demo` column already exists, just add a new row to `seed.sql` and run `supabase db reset` locally.

```sql
INSERT INTO public.content_items
  (id, tutor_id, type, title, body, subject_tag, language_tag, skill_level, status, is_demo)
VALUES
  (gen_random_uuid(), '00000000-0000-0000-0000-000000000001',
   'Post', 'New Demo Post', 'Content here...', 'Science', 'English', 'Intermediate', 'Published', true)
ON CONFLICT DO NOTHING;
```

### Workflow: Back-fill is_demo on existing seed rows

If seed data was inserted before the `is_demo` column existed, use the migration back-fill pattern:

```sql
UPDATE public.content_items
SET is_demo = true
WHERE title IN (
  -- enumerate the exact seed titles
  'Introduction to Algebra',
  'Python Crash Course'
)
AND is_demo = false;
```

Always enumerate titles explicitly — never use a wildcard UPDATE that could flag real content.

### Workflow: Strip demo data from a production database

```sql
-- Set is_demo = false on all rows (disables demo visibility)
UPDATE public.content_items SET is_demo = false WHERE is_demo = true;

-- Or hard-delete demo rows if they're clearly fake
DELETE FROM public.content_items
WHERE is_demo = true;
```

## Troubleshooting

### Demo content not showing for new tutors
**Cause:** The application query is not including the `OR is_demo = true` condition.
**Solution:** Audit all queries that fetch content for a specific tutor. Add the OR condition to each.

### Seed fails with "foreign key violation" on content_items
**Cause:** The demo tutor UUID doesn't exist in `public.users` when content is inserted.
**Solution:** The `handle_new_user` trigger creates the `public.users` row when the `auth.users` row is inserted. Ensure the `auth.users` insert runs first (and the trigger fires) before the content insert. In a `DO $$` block, use a small `PERFORM pg_sleep(0.1)` or restructure into separate statements if the trigger timing is an issue.

### `supabase db reset` wipes demo data
**Cause:** `db reset` re-runs all migrations and then `seed.sql`. This is correct behavior.
**Solution:** All persistent demo data must be in `seed.sql`, not applied via one-off SQL in the dashboard. Keep `seed.sql` as the authoritative source.

### Back-fill UPDATE marks real content as demo
**Cause:** A real content item happens to have the same title as a seed item.
**Solution:** Always add `AND is_demo = false` to back-fill UPDATEs, and prefer UUID-based targeting when possible.

## Best Practices

- The `is_demo` flag is the only supported mechanism for making seed content cross-tenant visible — do not change RLS policies for this purpose.
- Use `ON CONFLICT DO NOTHING` on all seed inserts to make them idempotent.
- Enumerate seed titles explicitly in back-fill migrations — no wildcards.
- Use a recognizable, fixed UUID range for demo users (`00000000-0000-0000-0000-00000000000X`).
- Never include real personal data in seed files.
- Seed files are only for development and CI. Add pipeline-level guards to prevent accidental production seeding.
- Keep `seed.sql` in version control alongside migrations — it is part of the schema definition for development environments.
