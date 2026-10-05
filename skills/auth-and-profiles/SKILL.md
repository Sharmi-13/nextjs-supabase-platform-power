---
name: "auth-and-profiles"
description: "Implement Supabase authentication, user profile creation, and the SECURITY DEFINER trigger pattern. Use when adding registration, login, logout, profile reads, or the public.users table to a Next.js + Supabase project."
license: "MIT"
metadata:
  author: "Link2Skill Team"
  version: "1.0.0"
---

# Authentication and User Profiles

## Overview

Supabase Auth manages passwords, sessions, email verification, and refresh tokens. Application code never touches these directly. The profile record in `public.users` is created atomically via a `SECURITY DEFINER` trigger on `auth.users`, which is the only safe pattern when email confirmation is enabled. This skill covers registration, login, logout, profile reads, and the trigger that glues Auth to the application database.

## Prerequisites Checklist

- [ ] `public.users` table exists and references `auth.users(id) ON DELETE CASCADE`
- [ ] `handle_new_user` trigger migration has been applied
- [ ] RLS is enabled on `public.users` (see `row-level-security` skill)
- [ ] Server Supabase client is set up (see `nextjs-supabase-setup` skill)
- [ ] Environment variables `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` are set

## The Trigger Pattern

### Why a trigger instead of a direct INSERT

When Supabase email confirmation is enabled, `signUp()` returns no session. Any attempt by the route handler to INSERT into `public.users` runs unauthenticated — RLS denies it.

A `SECURITY DEFINER` trigger on `auth.users` runs with the privileges of the function owner (`postgres`), bypassing RLS, and executes atomically inside the same transaction as the `auth.users` row creation.

```sql
-- migration: handle_new_user trigger
CREATE OR REPLACE FUNCTION public.handle_new_user()
  RETURNS TRIGGER
  LANGUAGE plpgsql
  SECURITY DEFINER
  SET search_path = public  -- pin to prevent search-path injection
AS $$
DECLARE
  v_username TEXT;
  v_roles    TEXT[];
BEGIN
  -- Read username from metadata passed by signUp()
  v_username := COALESCE(NEW.raw_user_meta_data->>'username', '');

  -- Read roles array from metadata; default to ['learner'] if absent
  v_roles := CASE
    WHEN NEW.raw_user_meta_data->'roles' IS NOT NULL
         AND jsonb_array_length(NEW.raw_user_meta_data->'roles') > 0
    THEN ARRAY(
      SELECT jsonb_array_elements_text(NEW.raw_user_meta_data->'roles')
    )
    ELSE ARRAY['learner']::TEXT[]
  END;

  INSERT INTO public.users (id, email, username, roles)
  VALUES (NEW.id, NEW.email, v_username, v_roles);

  RETURN NEW;
END;
$$;

DROP TRIGGER IF EXISTS on_auth_user_created ON auth.users;
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

**Rules for this trigger:**
- Do not remove `SET search_path = public` — it prevents search-path injection.
- Do not remove `SECURITY DEFINER` — the trigger cannot insert without it when email confirmation is on.
- Do not expand the trigger's scope beyond creating the `public.users` row.
- Do not grant the trigger function additional privileges.

## Step-by-Step Guide

### 1. Registration

Pass `username` and `roles` through `signUp()` metadata so the trigger can read them:

```typescript
// app/api/auth/register/route.ts
import { createClient } from "@/lib/supabase/server";
import { success, error } from "@/lib/api/response";
import { zodFieldErrors } from "@/lib/api/zod";
import { registerSchema } from "@/lib/auth/validation";

export async function POST(request: Request): Promise<Response> {
  let body: unknown;
  try {
    body = await request.json();
  } catch {
    return error("bad_request", "Request body must be valid JSON", 400);
  }

  const parsed = registerSchema.safeParse(body);
  if (!parsed.success) {
    return error("validation_failed", "One or more fields are invalid", 422,
      zodFieldErrors(parsed.error));
  }

  const { email, username, password, roles } = parsed.data;
  const supabase = await createClient();

  const { data: authData, error: authError } = await supabase.auth.signUp({
    email,
    password,
    options: {
      // username and roles are read by the handle_new_user trigger
      data: { username, roles },
    },
  });

  if (authError) {
    // Duplicate email — return generic 409; never disclose which field conflicted
    if (
      authError.code === "user_already_exists" ||
      authError.code === "email_exists" ||
      authError.message?.toLowerCase().includes("already registered")
    ) {
      return error("email_conflict", "An account with this email already exists", 409);
    }

    // Duplicate username — surfaced from the trigger's INSERT constraint
    if (authError.message?.includes("users_username_key")) {
      return error("username_conflict", "This username is already taken", 409);
    }

    console.error("[register] Auth error:", authError);
    return error("internal_error", "Registration failed. Please try again.", 500);
  }

  if (!authData.user) {
    return error("internal_error", "Registration failed. Please try again.", 500);
  }

  return success({
    id: authData.user.id,
    email,
    username,
    roles,
    emailConfirmationRequired: !authData.user.email_confirmed_at,
  }, 201);
}
```

### 2. Login

```typescript
// app/api/auth/login/route.ts
export async function POST(request: Request): Promise<Response> {
  let body: unknown;
  try {
    body = await request.json();
  } catch {
    return error("bad_request", "Request body must be valid JSON", 400);
  }

  const parsed = loginSchema.safeParse(body);
  if (!parsed.success) {
    return error("validation_failed", "One or more fields are invalid", 422,
      zodFieldErrors(parsed.error));
  }

  const { email, password } = parsed.data;
  const supabase = await createClient();

  const { data, error: authError } = await supabase.auth.signInWithPassword({
    email,
    password,
  });

  if (authError) {
    // Use a generic message for all login failures — never disclose which field was wrong
    return error("invalid_credentials", "Invalid email or password", 401);
  }

  return success({ id: data.user.id, email: data.user.email });
}
```

### 3. Logout

```typescript
// app/api/auth/logout/route.ts
export async function POST(_request: Request): Promise<Response> {
  const supabase = await createClient();
  await supabase.auth.signOut();
  return success({ message: "Logged out successfully" });
}
```

### 4. Read the authenticated user's profile

After authentication, resolve the application profile from `public.users`, not from `auth.users`. The `public.users` row contains application-level data (username, roles) that `auth.users` does not.

```typescript
// In any API route or Server Component
const supabase = await createClient();

// Step 1: get the auth user (verifies the session JWT)
const { data: { user }, error: authError } = await supabase.auth.getUser();
if (authError || !user) {
  return error("unauthorized", "Authentication required", 401);
}

// Step 2: fetch the application profile
const { data: profile, error: profileError } = await supabase
  .from("users")
  .select("id, email, username, roles, created_at")
  .eq("id", user.id)
  .single();

if (profileError || !profile) {
  return error("not_found", "User profile not found", 404);
}
```

### 5. Registration input validation

Define schemas in `lib/auth/validation.ts`, separate from route files so tests can import them without triggering Next.js server modules.

```typescript
// lib/auth/validation.ts
import { z } from "zod";

export const emailSchema = z
  .string()
  .email("A valid email address is required");

export const usernameSchema = z
  .string()
  .min(3, "Username must be at least 3 characters")
  .max(50, "Username must be at most 50 characters")
  .regex(/^[A-Za-z0-9_-]+$/, "Username may only contain letters, digits, underscores, and hyphens");

export const passwordSchema = z
  .string()
  .min(8, "Password must be at least 8 characters")
  .refine((v) => /[A-Z]/.test(v), { message: "Must contain at least one uppercase letter" })
  .refine((v) => /[a-z]/.test(v), { message: "Must contain at least one lowercase letter" })
  .refine((v) => /[0-9]/.test(v), { message: "Must contain at least one digit" });

export const rolesSchema = z
  .array(z.enum(["learner", "tutor"]))
  .min(1, "At least one role must be selected")
  .default(["learner"]);

export const registerSchema = z.object({
  email: emailSchema,
  username: usernameSchema,
  password: passwordSchema,
  roles: rolesSchema,
});
```

### 6. public.users table schema

```sql
CREATE TABLE users (
    id         UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    email      VARCHAR(255) UNIQUE NOT NULL,
    username   VARCHAR(50)  UNIQUE NOT NULL,
    roles      TEXT[] NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Key design decisions:
- `id` matches `auth.users.id` — UUID, not a serial. This is the primary key.
- `ON DELETE CASCADE` — deleting from `auth.users` cascades to `public.users`.
- `roles` is a `TEXT[]` — allows a user to hold multiple roles simultaneously (e.g. `['learner', 'tutor']`).
- Never INSERT into this table from application code.

## Common Workflows

### Workflow: Update a user's profile (username or roles)

```typescript
// app/api/user/profile/route.ts
export async function PUT(request: Request): Promise<Response> {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return error("unauthorized", "Authentication required", 401);

  const parsed = updateProfileSchema.safeParse(await request.json());
  if (!parsed.success) {
    return error("validation_failed", "Invalid input", 422, zodFieldErrors(parsed.error));
  }

  const { data, error: updateError } = await supabase
    .from("users")
    .update({ username: parsed.data.username, updated_at: new Date().toISOString() })
    .eq("id", user.id)
    .select()
    .single();

  if (updateError) {
    if (updateError.code === "23505") {  // unique_violation
      return error("username_conflict", "This username is already taken", 409);
    }
    return error("internal_error", "Update failed", 500);
  }

  return success(data);
}
```

### Workflow: Check if a user has a specific role

```typescript
// lib/auth/roles.ts
export function hasRole(roles: string[], role: string): boolean {
  return roles.includes(role);
}

// In a route handler
if (!hasRole(profile.roles, "tutor")) {
  return error("forbidden", "Tutor access required", 403);
}
```

## Troubleshooting

### Registration succeeds but public.users row is missing
**Cause:** The `handle_new_user` trigger is not installed, or it errored silently.
**Solution:** Verify the trigger exists: `SELECT * FROM pg_trigger WHERE tgname = 'on_auth_user_created'`. Check Supabase logs for trigger errors. Re-run the trigger migration if needed.

### Login returns 401 even with correct credentials
**Cause:** Email confirmation is pending, or the user registered before email confirmation was disabled.
**Solution:** Check the Supabase Auth dashboard for the user's confirmation status. For development, disable email confirmation in project settings.

### signUp() returns `user_already_exists` but the user doesn't appear in auth.users
**Cause:** Supabase has a soft-deleted user with the same email, or a partial registration that wasn't cleaned up.
**Solution:** Check Supabase Auth dashboard for deleted/unconfirmed users. Do not expose this detail to the client — always return generic 409.

### Trigger throws "null value in column username violates not-null constraint"
**Cause:** `signUp()` was called without passing `username` in `options.data`.
**Solution:** Always pass `options: { data: { username, roles } }` in every `signUp()` call.

## Best Practices

- Never manage passwords, sessions, or tokens manually — delegate everything to Supabase Auth.
- Always use `supabase.auth.getUser()` to verify identity in route handlers, not `getSession()` — `getUser()` re-validates the JWT with the server.
- Never expose auth error details to the client. Log them server-side.
- Always return generic messages for auth conflicts to prevent email enumeration.
- The `handle_new_user` trigger is the only correct way to create `public.users` rows — enforce this in code review.
- Keep `SECURITY DEFINER` scope minimal — only the specific INSERT the trigger needs.
- Store roles in `public.users`, not in `auth.users.raw_user_meta_data` — the metadata is writable by the client and not RLS-protected.
