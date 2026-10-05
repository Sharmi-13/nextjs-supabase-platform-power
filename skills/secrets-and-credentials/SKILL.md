---
name: "secrets-and-credentials"
description: "Protect API keys, Supabase credentials, and environment variables in a Next.js + Supabase project. Use when adding new environment variables, reviewing secret handling, setting up CI/CD, or auditing a codebase for leaked credentials."
license: "MIT"
metadata:
  author: "Link2Skill Team"
  version: "1.0.0"
---

# Secrets and Credentials

## Overview

Secrets that leak to the client, appear in logs, or get committed to version control are the most common and damaging security failures in web applications. This skill covers the rules for handling Supabase keys, environment variables, and sensitive data in a Next.js + Supabase project — drawn directly from the Link2Skill security model.

## Prerequisites Checklist

- [ ] `.env.local` is listed in `.gitignore`
- [ ] `@t3-oss/env-nextjs` is installed and used for all env var access
- [ ] The Supabase service role key is never in a `NEXT_PUBLIC_*` variable
- [ ] No `.env*` files other than `.env.local` (and `.env.example`) are committed

## The Two Supabase Keys

| Key | Variable name | Who can see it | Used in |
|-----|---------------|----------------|---------|
| Anon key | `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Public (client + server) | Browser client, server client |
| Service role key | `SUPABASE_SERVICE_ROLE_KEY` | Server only — never client | Seed scripts, admin jobs only |

The anon key is safe to expose. It is constrained by RLS. The service role key **bypasses RLS entirely** — it must never appear in any `NEXT_PUBLIC_*` variable, any client component, or any API response.

## Step-by-Step Guide

### 1. Store secrets in .env.local only

```bash
# .env.local — never commit this file
NEXT_PUBLIC_SUPABASE_URL=https://yourproject.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Provide a safe example file that documents required variables without values:

```bash
# .env.example — commit this file
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
```

### 2. Validate all env vars at startup with @t3-oss/env-nextjs

Never read `process.env` directly in application code. All environment variables must go through the validated schema.

```typescript
// lib/env/index.ts
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const env = createEnv({
  server: {
    // Server-only secrets — never prefixed with NEXT_PUBLIC_
    SUPABASE_SERVICE_ROLE_KEY: z.string().min(1),
  },
  client: {
    // Public vars — safe to expose to the browser
    NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
    NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(1),
  },
  runtimeEnv: {
    SUPABASE_SERVICE_ROLE_KEY: process.env.SUPABASE_SERVICE_ROLE_KEY,
    NEXT_PUBLIC_SUPABASE_URL: process.env.NEXT_PUBLIC_SUPABASE_URL,
    NEXT_PUBLIC_SUPABASE_ANON_KEY: process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
  },
});
```

If a variable is missing at startup, the app throws an error immediately rather than failing silently at runtime.

### 3. Never log secrets

```typescript
// ❌ Never do this
console.log("Supabase error:", authError);  // may contain tokens or emails
console.error("Config:", process.env);       // dumps all env vars

// ✅ Log only the fields you need
console.error("[login] Auth error code:", authError.code);
```

Specifically never log:
- Full `authError` objects (may contain email addresses)
- Full `dbError` objects (may contain row data)
- Request bodies for auth routes (contain passwords)
- Any variable read from `process.env`

### 4. Never expose Supabase internals in API responses

```typescript
// ❌ Never return internal error details
return Response.json({ error: authError.message });
return Response.json({ error: dbError.details });
return Response.json({ error: dbError.hint });

// ✅ Always return opaque, user-facing messages
return error("internal_error", "Registration failed. Please try again.", 500);
```

For auth conflicts (duplicate email, duplicate username), return generic messages:
```typescript
// ❌ Leaks which field caused the conflict — enables email enumeration
return error("conflict", "Email already in use", 409);

// ✅ Generic — does not confirm whether the email exists
return error("email_conflict", "An account with this email already exists", 409);
```

### 5. Gitignore rules

```gitignore
# .gitignore — secrets and local config
.env
.env.local
.env.*.local

# Supabase local config (contains service role key)
supabase/.branches
supabase/.temp

# Key files
*.pem
*.key
*.p12
*.pfx
*.jks
*.keystore
id_rsa
id_ed25519
*.crt
```

Never commit `.env.local`. If it is accidentally committed, treat the secrets as compromised and rotate all keys immediately.

### 6. Handle token and password storage

- **Password reset tokens** and **email verification tokens** are stored as HMAC hashes (`token_hash`), never as plaintext. This is handled by Supabase Auth internally — do not build custom token storage.
- **Passwords** are never stored or logged in application code. Supabase Auth handles bcrypt hashing.
- **Session tokens** are managed by `@supabase/ssr` via HTTP-only cookies. Never read or write session cookies manually.

### 7. Service role key usage

The service role key is only ever used in:
- Local seed scripts (`supabase/seed.sql` run via Supabase CLI)
- Server-side admin jobs that run outside the user request context
- Never in API routes, never in client components

```typescript
// lib/supabase/admin.ts — server-side only, never imported by client code
import { createClient } from "@supabase/supabase-js";
import { env } from "@/lib/env";

// Only use this client in admin/seed contexts — it bypasses RLS
export function createAdminClient() {
  return createClient(
    env.NEXT_PUBLIC_SUPABASE_URL,
    env.SUPABASE_SERVICE_ROLE_KEY,
  );
}
```

Add a comment to every file that imports this client to make the risk visible in code review.

### 8. SECURITY DEFINER functions

The `handle_new_user` trigger uses `SECURITY DEFINER` to bypass RLS. Keep these rules:

- Always pin `SET search_path = public` to prevent schema injection.
- The function's privilege scope must be the minimum needed — only the specific INSERT.
- Never expand a `SECURITY DEFINER` function to do work beyond its documented purpose.
- Document in migration comments why `SECURITY DEFINER` is required.

```sql
CREATE OR REPLACE FUNCTION public.handle_new_user()
  RETURNS TRIGGER
  LANGUAGE plpgsql
  SECURITY DEFINER
  SET search_path = public   -- ← required: prevents search-path injection
AS $$ ... $$;
```

### 9. SQL injection prevention

All database queries must use parameterized inputs via the Supabase client. Never concatenate user input into query strings.

```typescript
// ✅ Parameterized — safe
const { data } = await supabase
  .from("content_items")
  .select("*")
  .eq("tutor_id", tutorId)      // parameterized
  .eq("title", userInput);       // parameterized

// ❌ Never do this
const { data } = await supabase
  .rpc("raw_query", { sql: `SELECT * WHERE title = '${userInput}'` });
```

## Common Workflows

### Workflow: Audit for accidentally committed secrets

```bash
# Search for patterns that look like Supabase keys or tokens
git log --all --full-history -- "*.env*"
git grep -i "service_role" -- "*.ts" "*.tsx" "*.js"
git grep -i "supabase_key" -- "*.ts" "*.tsx"
git grep "eyJhbGci" -- "*.ts" "*.tsx" "*.json"  # JWT prefix
```

If a secret is found in git history:
1. Rotate the key immediately in the Supabase dashboard.
2. Use `git filter-repo` or BFG Repo Cleaner to remove from history.
3. Force-push and notify all collaborators to re-clone.

### Workflow: Add a new server-only environment variable

1. Add to `.env.local`: `MY_NEW_SECRET=value`
2. Add to `.env.example`: `MY_NEW_SECRET=`
3. Add to `lib/env/index.ts` server schema: `MY_NEW_SECRET: z.string().min(1)`
4. Add to `runtimeEnv`: `MY_NEW_SECRET: process.env.MY_NEW_SECRET`
5. Reference via `env.MY_NEW_SECRET` — never `process.env.MY_NEW_SECRET` directly.

### Workflow: Set up secrets in Vercel

Never paste secrets into source code for deployment. Use the Vercel dashboard or CLI:

```bash
# Vercel CLI
vercel env add SUPABASE_SERVICE_ROLE_KEY production
vercel env add NEXT_PUBLIC_SUPABASE_URL production
vercel env add NEXT_PUBLIC_SUPABASE_ANON_KEY production
```

Then pull to local for development:
```bash
vercel env pull .env.local
```

## Troubleshooting

### Build fails with "Missing environment variable: SUPABASE_SERVICE_ROLE_KEY"
**Cause:** `@t3-oss/env-nextjs` validates at build time. The variable is not set in the build environment.
**Solution:** Add the variable to the CI/CD environment (GitHub Actions secrets, Vercel env, etc.). For local builds, ensure `.env.local` exists and contains the variable.

### "NEXT_PUBLIC_ variables can't be used in server-side code"
**Cause:** This is a misunderstanding — `NEXT_PUBLIC_` variables ARE available server-side. The restriction is the reverse: server-only variables (without the prefix) are NOT available client-side.
**Solution:** Use `NEXT_PUBLIC_` for anything the browser needs. Omit the prefix for server-only secrets.

### Auth error object appears in Supabase logs
**Cause:** `console.error(authError)` in an API route.
**Solution:** Log only `authError.code` and `authError.status`, not the full object.

## Best Practices

- Treat every `.env*` file as containing secrets — `.gitignore` all of them except `.env.example`.
- Rotate keys immediately if any secret appears in git history or logs.
- The service role key is a master key — handle it with the same care as a production database password.
- Never return Supabase error details to API clients.
- Log error codes for debugging, never full error objects or request bodies for auth routes.
- `SECURITY DEFINER` functions are the only acceptable RLS bypass — document every instance.
- Run `git grep "eyJhbGci"` regularly to catch accidentally committed JWT tokens.
