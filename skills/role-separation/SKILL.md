---
name: "role-separation"
description: "Implement Tutor and Learner role separation in a Next.js + Supabase learning platform. Use when routing users by role, protecting tutor-only or learner-only routes, building role-specific dashboards, or enforcing role checks in API routes."
license: "MIT"
metadata:
  author: "Link2Skill Team"
  version: "1.0.0"
---

# Tutor and Learner Role Separation

## Overview

Link2Skill-style platforms have two primary user roles: **Tutor** (creates and publishes content) and **Learner** (discovers and consumes content). A single user can hold both roles simultaneously. Roles are stored as a `TEXT[]` array in `public.users.roles` and are the authoritative source for all role-based access decisions. This skill covers how to store, check, and enforce roles across routes, components, and the database.

## Prerequisites Checklist

- [ ] `public.users` table has a `roles TEXT[]` column
- [ ] `handle_new_user` trigger populates roles from `signUp()` metadata
- [ ] RLS policies are in place (see `row-level-security` skill)
- [ ] All role enums are centralized in `lib/types/index.ts`

## Role Model

### Data shape

```typescript
// lib/types/index.ts — centralized enums, never redefined elsewhere
export enum UserRole {
  LEARNER = "learner",
  TUTOR   = "tutor",
}
```

```sql
-- In public.users
roles TEXT[] NOT NULL DEFAULT '{}'
-- Example values: '{learner}', '{tutor}', '{learner,tutor}'
```

### Key design decisions

- **Array, not a single column.** A user can be both learner and tutor without a second account.
- **Stored in `public.users`, not in `auth.users.raw_user_meta_data`.** Metadata is client-writable and bypasses RLS. `public.users` is RLS-protected.
- **Role is set at registration** by the client, defaulting to `['learner']` if not provided.
- **Role changes go through the API**, not direct table updates from the client.

## Step-by-Step Guide

### 1. Set roles at registration

The client passes the desired roles during `signUp()`. The trigger reads them from `raw_user_meta_data`.

```typescript
// Client-side registration call
const { data, error } = await supabase.auth.signUp({
  email,
  password,
  options: {
    data: {
      username,
      roles: ["learner"],         // or ["tutor"] or ["learner", "tutor"]
    },
  },
});
```

The trigger defaults to `['learner']` if roles are absent or empty:
```sql
v_roles := CASE
  WHEN NEW.raw_user_meta_data->'roles' IS NOT NULL
       AND jsonb_array_length(NEW.raw_user_meta_data->'roles') > 0
  THEN ARRAY(SELECT jsonb_array_elements_text(NEW.raw_user_meta_data->'roles'))
  ELSE ARRAY['learner']::TEXT[]
END;
```

### 2. Centralize role checking logic

```typescript
// lib/auth/roles.ts
import { UserRole } from "@/lib/types";

export function hasRole(roles: string[], role: UserRole): boolean {
  return roles.includes(role);
}

export function isTutor(roles: string[]): boolean {
  return hasRole(roles, UserRole.TUTOR);
}

export function isLearner(roles: string[]): boolean {
  return hasRole(roles, UserRole.LEARNER);
}

export function requireRole(
  roles: string[],
  role: UserRole,
): { allowed: boolean } {
  return { allowed: hasRole(roles, role) };
}
```

### 3. Enforce role checks in API routes

Always fetch the profile from `public.users` to get the authoritative roles. Never trust `auth.users.raw_user_meta_data` for authorization decisions.

```typescript
// Pattern used in tutor-only routes
export async function POST(request: Request): Promise<Response> {
  const supabase = await createClient();

  // Step 1: verify identity
  const { data: { user }, error: authError } = await supabase.auth.getUser();
  if (authError || !user) {
    return error("unauthorized", "Authentication required", 401);
  }

  // Step 2: fetch authoritative profile with roles
  const { data: profile } = await supabase
    .from("users")
    .select("id, roles")
    .eq("id", user.id)
    .single();

  if (!profile) {
    return error("not_found", "User profile not found", 404);
  }

  // Step 3: enforce role
  if (!isTutor(profile.roles)) {
    return error("forbidden", "Tutor access required", 403);
  }

  // ... tutor-specific logic
}
```

### 4. Role-based dashboard routing

Redirect users to the correct dashboard based on their roles after login.

```typescript
// app/dashboard/page.tsx (root dashboard redirect)
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";

export default async function DashboardPage() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) redirect("/login");

  const { data: profile } = await supabase
    .from("users")
    .select("roles")
    .eq("id", user.id)
    .single();

  if (!profile) redirect("/login");

  // Tutors get priority if user holds both roles
  if (profile.roles.includes("tutor")) {
    redirect("/dashboard/tutor");
  }

  redirect("/dashboard/learner");
}
```

### 5. Role-specific layouts

Use Next.js layout files to protect entire dashboard subtrees.

```typescript
// app/dashboard/tutor/layout.tsx
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";
import { TutorSidebar } from "@/app/components/tutor/TutorSidebar";

export default async function TutorLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) redirect("/login");

  const { data: profile } = await supabase
    .from("users")
    .select("roles")
    .eq("id", user.id)
    .single();

  if (!profile?.roles.includes("tutor")) {
    redirect("/dashboard/learner");
  }

  return (
    <div className="flex">
      <TutorSidebar />
      <main className="flex-1">{children}</main>
    </div>
  );
}
```

```typescript
// app/dashboard/learner/layout.tsx — same pattern for learner role
```

### 6. RLS role enforcement

RLS policies enforce ownership at the database level. Application-layer role checks are an additional defence but RLS is the primary safeguard.

```sql
-- Only the owning tutor can INSERT their own content
CREATE POLICY "content_items_insert_own"
  ON public.content_items
  FOR INSERT
  WITH CHECK (auth.uid() = tutor_id);
```

Note: RLS cannot check the `roles` column in a performant way for most policies. Use RLS for ownership (`auth.uid() = owner_id`) and handle role-based business logic in the application layer.

### 7. Separate route trees per role

Organize the file system to match role boundaries:

```
app/
  dashboard/
    tutor/
      layout.tsx          ← role guard lives here
      page.tsx            ← tutor home
      content/page.tsx    ← content management
      community/page.tsx
      notifications/page.tsx
      profile/page.tsx
      settings/page.tsx
    learner/
      layout.tsx          ← role guard lives here
      page.tsx            ← learner home
      discover/page.tsx
      following/page.tsx
      saved/page.tsx
      notifications/page.tsx
      profile/page.tsx
  api/
    tutor/                ← tutor-only endpoints
      content/route.ts
      settings/route.ts
    user/                 ← shared endpoints
      profile/route.ts
```

### 8. Component-level role rendering

For pages accessible to both roles but with different content:

```typescript
"use client";

interface ProfileActionsProps {
  roles: string[];
}

export function ProfileActions({ roles }: ProfileActionsProps) {
  const userIsTutor = roles.includes("tutor");
  const userIsLearner = roles.includes("learner");

  return (
    <div>
      {userIsTutor && (
        <a href="/dashboard/tutor">Tutor Dashboard</a>
      )}
      {userIsLearner && (
        <a href="/dashboard/learner">Learner Dashboard</a>
      )}
    </div>
  );
}
```

## Common Workflows

### Workflow: Add a new tutor-only API route

1. Create the route file at `app/api/tutor/<action>/route.ts`.
2. Start every handler with the auth + profile + role check pattern from section 3.
3. Return `403 forbidden` (not 404) when a learner attempts to reach a tutor route — 404 would hide the existence of the endpoint.

### Workflow: Allow a user to become a tutor (role upgrade)

```typescript
// app/api/user/settings/route.ts
export async function PUT(request: Request): Promise<Response> {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return error("unauthorized", "Authentication required", 401);

  const parsed = settingsSchema.safeParse(await request.json());
  if (!parsed.success) {
    return error("validation_failed", "Invalid input", 422, zodFieldErrors(parsed.error));
  }

  // Merge new roles with existing ones — never remove roles without explicit intent
  const { data: current } = await supabase
    .from("users").select("roles").eq("id", user.id).single();

  const updatedRoles = [...new Set([...(current?.roles ?? []), ...parsed.data.roles])];

  const { data, error: updateError } = await supabase
    .from("users")
    .update({ roles: updatedRoles, updated_at: new Date().toISOString() })
    .eq("id", user.id)
    .select()
    .single();

  if (updateError) return error("internal_error", "Settings update failed", 500);

  return success(data);
}
```

## Troubleshooting

### 403 returned even though user has the correct role
**Cause:** Role check is reading from `auth.users.raw_user_meta_data` instead of `public.users.roles`. Metadata can become stale if roles are updated.
**Solution:** Always read roles from `public.users`, never from the JWT claims or metadata.

### User stuck on learner dashboard after being granted tutor role
**Cause:** Session/JWT cached the old roles. The redirect in the layout reads fresh DB data, but a stale client-side cache might be displaying old UI.
**Solution:** Force a page reload after a role change. The server layout will re-check from `public.users` on every navigation.

### Both roles redirecting to the same dashboard
**Cause:** The redirect logic in the root dashboard page doesn't check both roles.
**Solution:** Check `roles.includes("tutor")` first (tutor priority), then fall back to `roles.includes("learner")`.

## Best Practices

- Roles live in `public.users.roles` — this is the single source of truth.
- Never read roles from `auth.users.raw_user_meta_data` for authorization.
- The layout file is the correct place for route-level role guards — not individual page files.
- Use 403 (forbidden) when a user is authenticated but lacks the role; use 401 when unauthenticated.
- Keep all role enums in `lib/types/index.ts` — no magic strings scattered through the codebase.
- The application layer enforces roles; RLS enforces ownership. Both layers are needed.
