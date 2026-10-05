---
name: "nextjs-supabase-setup"
description: "Configure Next.js App Router with Supabase for server and client components. Use when wiring up the Supabase client, environment variables, API routes, or response envelopes in a Next.js 16+ project."
license: "MIT"
metadata:
  author: "Link2Skill Team"
  version: "1.0.0"
---

# Next.js + Supabase Setup

## Overview

This skill covers the integration layer between Next.js App Router and Supabase: the two-client pattern (browser vs server), type-safe environment variables, API route conventions, and the shared response envelope. Patterns are drawn from the Link2Skill codebase and apply to any Next.js 16+ App Router project using `@supabase/ssr`.

## Prerequisites Checklist

- [ ] `@supabase/ssr` installed (`npm install @supabase/ssr`)
- [ ] `@t3-oss/env-nextjs` installed for type-safe env vars
- [ ] `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` set in `.env.local`
- [ ] Next.js App Router project (files under `app/`)
- [ ] TypeScript strict mode enabled (`"strict": true` in `tsconfig.json`)

## The Two-Client Rule

Never use a single Supabase client everywhere. The browser client and server client have fundamentally different cookie access:

| Client | File | Used in |
|--------|------|---------|
| Browser client | `lib/supabase/client.ts` | `"use client"` components only |
| Server client | `lib/supabase/server.ts` | API routes, Server Components, Server Actions |

Using the browser client in a server context loses the user's session. Using the server client in a browser context causes runtime errors because `cookies()` is not available.

## Step-by-Step Guide

### 1. Create the browser client

```typescript
// lib/supabase/client.ts
import { createBrowserClient } from "@supabase/ssr";
import { env } from "@/lib/env";

export function createClient() {
  return createBrowserClient(
    env.NEXT_PUBLIC_SUPABASE_URL,
    env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
  );
}
```

### 2. Create the server client

The server client must be `async` because `cookies()` from `next/headers` is async in Next.js 15+.

```typescript
// lib/supabase/server.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";
import { env } from "@/lib/env";

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    env.NEXT_PUBLIC_SUPABASE_URL,
    env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options),
            );
          } catch {
            // Server Components cannot modify cookies — safe to ignore.
          }
        },
      },
    },
  );
}
```

### 3. Set up type-safe environment variables

Never call `process.env` directly in application code. Use `@t3-oss/env-nextjs` to validate at startup.

```typescript
// lib/env/index.ts
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const env = createEnv({
  client: {
    NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
    NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(1),
  },
  runtimeEnv: {
    NEXT_PUBLIC_SUPABASE_URL: process.env.NEXT_PUBLIC_SUPABASE_URL,
    NEXT_PUBLIC_SUPABASE_ANON_KEY: process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
  },
});
```

Add new environment variables here, not as bare `process.env` references scattered through the codebase.

### 4. Structure API routes

All API routes live at `app/api/<domain>/<action>/route.ts` and export named HTTP verb handlers.

```typescript
// app/api/auth/login/route.ts
import { createClient } from "@/lib/supabase/server";
import { success, error } from "@/lib/api/response";

export async function POST(request: Request): Promise<Response> {
  // Always parse JSON defensively
  let body: unknown;
  try {
    body = await request.json();
  } catch {
    return error("bad_request", "Request body must be valid JSON", 400);
  }

  // Validate with Zod — never .parse(), always .safeParse()
  const parsed = loginSchema.safeParse(body);
  if (!parsed.success) {
    return error("validation_failed", "One or more fields are invalid", 422,
      zodFieldErrors(parsed.error));
  }

  const supabase = await createClient();
  // ... handler logic
}
```

### 5. Use the shared response envelope

Every API route must return responses through the shared helpers in `lib/api/response.ts`. Never use raw `Response.json()` calls in route handlers.

```typescript
// lib/api/response.ts

export interface FieldError {
  field: string;
  message: string;
}

/** Wraps payload in { data: T }. Defaults to HTTP 200. */
export function success<T>(data: T, status = 200): Response {
  return Response.json({ data }, { status });
}

/** Wraps in { error: { code, message, fields? } }. Omits fields when empty. */
export function error(
  code: string,
  message: string,
  status: number,
  fields?: FieldError[],
): Response {
  const body = {
    error: { code, message, ...(fields?.length ? { fields } : {}) },
  };
  return Response.json(body, { status });
}
```

HTTP status conventions:

| Status | Meaning |
|--------|---------|
| 200 | Success (GET, PUT, PATCH) |
| 201 | Created (POST) |
| 400 | Bad JSON |
| 401 | Unauthenticated |
| 403 | Forbidden (authenticated but not authorized) |
| 404 | Not found |
| 409 | Conflict (duplicate) |
| 422 | Validation failure |
| 500 | Internal server error |

### 6. Prefer Server Components

Default to Server Components. Only add `"use client"` when a component genuinely needs browser APIs, event handlers, or React state/effects.

```typescript
// Server Component — no directive needed
// app/dashboard/tutor/page.tsx
import { createClient } from "@/lib/supabase/server";

export default async function TutorDashboardPage() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  // ...
}
```

```typescript
// Client Component — only when interactivity is required
"use client";
// app/components/tutor/CreateContentForm.tsx
import { createClient } from "@/lib/supabase/client";
```

### 7. Path aliases

Use `@/` for all internal imports. Never use relative `../../` paths that cross directory boundaries.

```typescript
// ✅ Correct
import { success } from "@/lib/api/response";
import { createClient } from "@/lib/supabase/server";

// ❌ Avoid
import { success } from "../../../lib/api/response";
```

Configure in `tsconfig.json`:
```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./*"]
    }
  }
}
```

## Common Workflows

### Workflow: Read the authenticated user in an API route

```typescript
export async function GET(_request: Request): Promise<Response> {
  const supabase = await createClient();
  const { data: { user }, error: authError } = await supabase.auth.getUser();

  if (authError || !user) {
    return error("unauthorized", "Authentication required", 401);
  }

  // user.id is the auth.uid() — use it for RLS-guarded queries
  const { data, error: dbError } = await supabase
    .from("users")
    .select("*")
    .eq("id", user.id)
    .single();

  if (dbError || !data) {
    return error("not_found", "User not found", 404);
  }

  return success(data);
}
```

### Workflow: Validate Zod schemas in isolation from Next.js

Define Zod schemas in `lib/<domain>/validation.ts`, separate from the route handler. This lets tests import schemas without pulling in `next/headers` or `next/server`.

```typescript
// lib/auth/validation.ts  ← pure Zod, no Next.js imports
export const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
});

// app/api/auth/login/route.ts  ← imports schema from lib/
import { loginSchema } from "@/lib/auth/validation";
```

## Troubleshooting

### Error: "cookies() is not available in this context"
**Cause:** Server client being created at module scope (outside a request handler) or in a browser component.
**Solution:** Always call `await createClient()` inside the handler function, never at the top level.

### Error: "Cannot read properties of undefined (reading 'getAll')"
**Cause:** Forgetting `await` on the async server client factory.
**Solution:** Use `const supabase = await createClient()`, not `const supabase = createClient()`.

### Error: Missing environment variable at startup
**Cause:** Variable not added to the `@t3-oss/env-nextjs` schema, or not present in `.env.local`.
**Solution:** Add to `lib/env/index.ts` schema AND to `.env.local`. Never skip the schema — it's the only layer that validates at boot time.

### Supabase returns no session after `signUp()`
**Cause:** Email confirmation is enabled in the Supabase project. `signUp()` sends a confirmation email and returns no session until confirmed.
**Solution:** Check `authData.user.email_confirmed_at`. If null, return `emailConfirmationRequired: true` in the response so the client can show the appropriate UI.

## Best Practices

- Always `await createClient()` on the server — it's async.
- Never share a single client instance across requests; create a fresh one per handler.
- Keep Zod schemas in `lib/`, not inside route files — tests cannot import route files that reference `next/headers`.
- Never expose Supabase error internals (`authError.message`, `dbError.details`) in API responses. Log server-side, return opaque messages.
- The service role key bypasses RLS entirely — it must never appear in client-side code or `NEXT_PUBLIC_*` variables.
