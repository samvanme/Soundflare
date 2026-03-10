# Phase 1: Database & Auth Foundation - Research

**Researched:** 2026-03-10
**Domain:** PostgreSQL RPC functions, Supabase RLS, Next.js 15 server-side auth guards
**Confidence:** HIGH

## Summary

Phase 1 builds the data layer and auth guards that Phase 2 (dashboard UI) and Phase 3 (Slack notifications) depend on. The codebase already has a self-hosted Supabase stack (PostgreSQL 15 + GoTrue + PostgREST + Kong), a working auth flow via `@supabase/ssr`, and a service-role admin client (`getSupabaseAdmin()` in `src/lib/supabase-server.ts`). The existing `PYPE_ADMINS` env var pattern is already used in `src/app/api/email/notify-admins/route.ts` for admin email identification.

The work breaks into three areas: (1) two PostgreSQL RPC functions (`agency_metrics`, `agency_failures`) with SECURITY DEFINER and restricted execution privileges, (2) two new tables (`slack_notification_configs`, `slack_notification_log`) with RLS enabled and no public policies, and (3) a Next.js auth guard pattern at the layout level for `/agency` plus a shared `assertAgencyAdmin()` helper for API routes.

**Primary recommendation:** Follow existing codebase patterns exactly -- SQL migrations in `database/`, service-role client via `getSupabaseAdmin()`, auth checks via `src/lib/auth.ts` patterns. No new libraries needed.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Agency admins identified via `PYPE_ADMINS` environment variable -- comma-separated email allowlist
- No new database table for admin identity; env var is sufficient for internal team
- Non-admin authenticated users hitting `/agency` are silently redirected to home (`/`)
- Unauthenticated users hitting `/agency` are redirected to `/sign-in?next=/agency` (return URL preserved)
- Agency metrics aggregate across ALL organizations -- no filtering or exclusion
- Each agent row includes its organization name (from `soundflare_projects.name`)
- Date granularity is daily (one row per agent per day)
- One webhook URL per agent (`UNIQUE(agent_id)` constraint on `slack_notification_configs`)
- `is_enabled` boolean toggle per config
- Throttle window hardcoded in code, not per-agent column
- `slack_notification_log` tracks: config_id, agent_id, call_log_ids (UUID array), status ('sent'/'failed'), error_message, sent_at
- Both tables CASCADE delete when parent agent is deleted
- Layout-level check in `/agency/layout.tsx` (server component)
- API routes under `/api/agency/*` check via shared `assertAgencyAdmin()` helper
- Middleware stays as-is (session refresh only)
- RLS on both notification tables: enabled with no anon/authenticated policies (service role only)
- RPC functions: SECURITY DEFINER with EXECUTE revoked from public/anon -- callable only via service role

### Claude's Discretion
- Exact RPC function signatures and return types
- Index strategy for metrics queries
- Throttle window duration (suggested ~5 minutes)
- Error type classification logic within the failures RPC
- SQL migration file organization

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| INFR-01 | Dashboard route restricted to admin users (PYPE_ADMINS list) | Auth guard pattern in layout.tsx + assertAgencyAdmin() helper; existing PYPE_ADMINS usage found in codebase |
| INFR-02 | Dashboard metrics computed via PostgreSQL RPC functions, not client-side | agency_metrics and agency_failures RPC functions; existing schema tables confirmed (call_reviews, soundflare_call_logs, soundflare_agents, soundflare_projects) |
| INFR-03 | Slack webhook URLs stored securely, accessed only through server-side API routes | slack_notification_configs table with RLS (no public policies); service-role-only access via getSupabaseAdmin() |
</phase_requirements>

## Standard Stack

### Core (already in project)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @supabase/supabase-js | ^2.51.0 | Service-role DB client | Already used via `getSupabaseAdmin()` in `src/lib/supabase-server.ts` |
| @supabase/ssr | ^0.6.1 | Server/client cookie-based auth | Already used in middleware and server client |
| next | ^15.4.8 | App router, server components, API routes | Project framework |
| supabase/postgres | 15.1.1.2 | PostgreSQL with Supabase extensions | Docker image already in docker-compose |

### Supporting (no new installs)
| Library | Purpose | When to Use |
|---------|---------|-------------|
| date-fns (^4.1.0) | Date formatting if needed in auth helpers | Already installed |

**Installation:** No new packages required. All dependencies exist.

## Architecture Patterns

### Project Structure (new files)
```
src/
├── app/
│   ├── agency/
│   │   └── layout.tsx           # Server component: auth + admin guard
│   └── api/
│       └── agency/
│           ├── metrics/route.ts  # GET: calls agency_metrics RPC
│           └── failures/route.ts # GET: calls agency_failures RPC
├── lib/
│   └── agency-auth.ts           # assertAgencyAdmin() shared helper
database/
├── add_agency_rpc_functions.sql  # agency_metrics + agency_failures RPCs
└── add_slack_notification_tables.sql # notification config + log tables
```

### Pattern 1: Admin Auth Guard (Layout-Level)
**What:** Server component layout that checks authentication then admin authorization before rendering children.
**When to use:** `/agency` route group.
**Example:**
```typescript
// src/app/agency/layout.tsx (server component)
import { redirect } from 'next/navigation'
import { createClient } from '@/utils/supabase/server'

export default async function AgencyLayout({ children }: { children: React.ReactNode }) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user) {
    redirect('/sign-in?next=/agency')
  }

  const adminEmails = (process.env.PYPE_ADMINS || '').split(',').map(e => e.trim().toLowerCase())
  if (!user.email || !adminEmails.includes(user.email.toLowerCase())) {
    redirect('/')
  }

  return <>{children}</>
}
```
**Source:** Follows existing pattern from `src/lib/auth.ts` (uses `supabase.auth.getUser()`) and `src/app/api/email/notify-admins/route.ts` (parses `PYPE_ADMINS`).

### Pattern 2: API Route Admin Guard (Shared Helper)
**What:** Reusable function that throws or returns user after verifying admin status.
**When to use:** All `/api/agency/*` route handlers.
**Example:**
```typescript
// src/lib/agency-auth.ts
import { createClient } from '@/utils/supabase/server'

export class UnauthorizedError extends Error {
  status: number
  constructor(message: string, status = 401) {
    super(message)
    this.status = status
  }
}

export async function assertAgencyAdmin() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user) {
    throw new UnauthorizedError('Not authenticated', 401)
  }

  const adminEmails = (process.env.PYPE_ADMINS || '').split(',').map(e => e.trim().toLowerCase())
  if (!user.email || !adminEmails.includes(user.email.toLowerCase())) {
    throw new UnauthorizedError('Not authorized', 403)
  }

  return user
}
```

### Pattern 3: Service-Role RPC Calls
**What:** API routes call RPC functions via the service-role client (bypasses RLS).
**When to use:** All agency data queries -- never expose these via browser client.
**Example:**
```typescript
// In API route handler
import { getSupabaseAdmin } from '@/lib/supabase-server'

const supabase = getSupabaseAdmin()
const { data, error } = await supabase.rpc('agency_metrics', {
  start_date: '2026-01-01',
  end_date: '2026-03-10'
})
```
**Source:** `getSupabaseAdmin()` already exists in `src/lib/supabase-server.ts`, uses `SUPABASE_SERVICE_ROLE_KEY`.

### Pattern 4: SQL Migration Files
**What:** Individual SQL files in `database/` directory, following existing naming convention.
**When to use:** All schema changes.
**Source:** Existing pattern: `database/add_call_reviews_table.sql`, `database/add_ai_review_trigger.sql`, etc.

### Anti-Patterns to Avoid
- **Client-side RPC calls for agency data:** Never use the browser Supabase client (`src/utils/supabase/client.ts`) for agency queries. All agency data must flow through server-side API routes using `getSupabaseAdmin()`.
- **Admin logic in middleware:** The middleware (`src/middleware.ts`) only handles session refresh. Admin checks belong in the layout and API route helpers, not middleware.
- **Granting EXECUTE to anon/authenticated on agency RPCs:** These functions must only be callable via service_role.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Auth session management | Custom JWT parsing | `supabase.auth.getUser()` via `createClient()` | Already handles cookie-based sessions, token refresh |
| Service-role DB access | Direct PostgreSQL connection | `getSupabaseAdmin()` from `src/lib/supabase-server.ts` | Lazy-initialized singleton, consistent URL handling |
| Admin email parsing | New admin table/roles system | `process.env.PYPE_ADMINS` split on commas | Locked decision; existing pattern in notify-admins route |
| Daily aggregation in JS | Client-side reduce/groupBy | PostgreSQL `DATE_TRUNC` + `GROUP BY` in RPC | Performance, consistency, data stays in DB |

## Common Pitfalls

### Pitfall 1: RPC Functions Not Accessible via PostgREST
**What goes wrong:** Creating an RPC function in `public` schema but PostgREST can't find it.
**Why it happens:** PostgREST only exposes functions in schemas listed in `PGRST_DB_SCHEMA`. Current config: `public,auth`.
**How to avoid:** Create functions in `public` schema (which is already exposed). Verify function exists after migration with `SELECT proname FROM pg_proc WHERE proname = 'agency_metrics'`.
**Warning signs:** `404` from PostgREST `/rest/v1/rpc/agency_metrics`.

### Pitfall 2: SECURITY DEFINER Runs as Function Owner
**What goes wrong:** SECURITY DEFINER function runs with the privileges of the user who created it (typically `postgres`), which bypasses RLS.
**Why it happens:** This is by design -- it's actually what we want here since agency queries need cross-organization access.
**How to avoid:** This is correct behavior for our use case. But also revoke EXECUTE from `anon` and `authenticated` to prevent direct calls. Only `service_role` (which bypasses RLS anyway) should call these.
**Warning signs:** If anon users can call the RPC directly via PostgREST.

### Pitfall 3: Default Privileges Don't Apply Retroactively
**What goes wrong:** `ALTER DEFAULT PRIVILEGES` in `setup-supabase.sql` grants future tables to `authenticated`/`service_role`, but new tables created in a migration might not get these grants if the migration runs as a different role.
**Why it happens:** Default privileges are role-specific. The migration must include explicit `GRANT` statements.
**How to avoid:** Every migration SQL file that creates tables or functions must include its own `GRANT` and `REVOKE` statements. Don't rely on default privileges.
**Warning signs:** `permission denied for table slack_notification_configs` errors.

### Pitfall 4: PostgREST Schema Cache
**What goes wrong:** After running a migration that adds new RPC functions, PostgREST returns 404.
**Why it happens:** PostgREST caches the schema on startup.
**How to avoid:** After running migrations, restart the `rest` container: `docker compose restart rest`. Also restart `kong` after any container restart (per CLAUDE.md notes).
**Warning signs:** New RPC returns 404 but `SELECT` from psql works fine.

### Pitfall 5: PYPE_ADMINS Not in .env.docker
**What goes wrong:** `process.env.PYPE_ADMINS` returns `undefined` in production container.
**Why it happens:** The env var exists in the notify-admins code but is not currently in `.env.docker`.
**How to avoid:** Add `PYPE_ADMINS=admin@soundflare.ai` (or correct emails) to `.env.docker`. The web container reads from `.env.docker` via `env_file` in docker-compose.
**Warning signs:** All users silently redirected from `/agency`, even admins.

### Pitfall 6: UUID Array Column Type
**What goes wrong:** Using `text[]` instead of `uuid[]` for `call_log_ids` in `slack_notification_log`.
**Why it happens:** Forgetting that call_log IDs are UUIDs.
**How to avoid:** Use `uuid[]` type for `call_log_ids` column in `slack_notification_log`.

## Code Examples

### RPC Function: agency_metrics
```sql
-- Source: Based on existing schema (soundflare_call_logs, soundflare_agents, soundflare_projects, call_reviews)
CREATE OR REPLACE FUNCTION public.agency_metrics(
  start_date date DEFAULT (CURRENT_DATE - INTERVAL '30 days')::date,
  end_date date DEFAULT CURRENT_DATE
)
RETURNS TABLE (
  agent_id uuid,
  agent_name varchar,
  organization_name varchar,
  call_date date,
  total_calls bigint,
  total_minutes numeric,
  failure_count bigint,
  failure_rate numeric
)
LANGUAGE sql
SECURITY DEFINER
STABLE
AS $$
  SELECT
    a.id AS agent_id,
    a.name AS agent_name,
    p.name AS organization_name,
    DATE_TRUNC('day', cl.created_at)::date AS call_date,
    COUNT(cl.id) AS total_calls,
    COALESCE(SUM(cl.duration_seconds), 0) / 60.0 AS total_minutes,
    COUNT(cr.id) FILTER (WHERE cr.error_count > 0) AS failure_count,
    CASE
      WHEN COUNT(cl.id) > 0
      THEN ROUND(COUNT(cr.id) FILTER (WHERE cr.error_count > 0)::numeric / COUNT(cl.id) * 100, 2)
      ELSE 0
    END AS failure_rate
  FROM soundflare_agents a
  JOIN soundflare_projects p ON a.project_id = p.id
  LEFT JOIN soundflare_call_logs cl ON cl.agent_id = a.id
    AND cl.created_at >= start_date
    AND cl.created_at < (end_date + INTERVAL '1 day')
  LEFT JOIN call_reviews cr ON cr.call_log_id = cl.id
    AND cr.status = 'completed'
  GROUP BY a.id, a.name, p.name, DATE_TRUNC('day', cl.created_at)::date
  ORDER BY call_date DESC NULLS LAST, a.name;
$$;

-- Restrict execution: only service_role can call this
REVOKE EXECUTE ON FUNCTION public.agency_metrics(date, date) FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION public.agency_metrics(date, date) FROM anon;
REVOKE EXECUTE ON FUNCTION public.agency_metrics(date, date) FROM authenticated;
GRANT EXECUTE ON FUNCTION public.agency_metrics(date, date) TO service_role;
```

### RPC Function: agency_failures
```sql
CREATE OR REPLACE FUNCTION public.agency_failures(
  start_date date DEFAULT (CURRENT_DATE - INTERVAL '30 days')::date,
  end_date date DEFAULT CURRENT_DATE,
  error_type text DEFAULT NULL
)
RETURNS TABLE (
  call_log_id uuid,
  call_id varchar,
  agent_id uuid,
  agent_name varchar,
  organization_name varchar,
  call_started_at timestamptz,
  duration_seconds integer,
  error_classification text,
  has_api_failures boolean,
  has_wrong_actions boolean,
  has_wrong_outputs boolean,
  error_count integer
)
LANGUAGE sql
SECURITY DEFINER
STABLE
AS $$
  SELECT
    cl.id AS call_log_id,
    cl.call_id,
    a.id AS agent_id,
    a.name AS agent_name,
    p.name AS organization_name,
    cl.call_started_at,
    cl.duration_seconds,
    CASE
      WHEN cr.has_api_failures AND cr.has_wrong_actions THEN 'MULTIPLE'
      WHEN cr.has_api_failures AND cr.has_wrong_outputs THEN 'MULTIPLE'
      WHEN cr.has_wrong_actions AND cr.has_wrong_outputs THEN 'MULTIPLE'
      WHEN cr.has_api_failures THEN 'API_FAILURE'
      WHEN cr.has_wrong_actions THEN 'WRONG_ACTION'
      WHEN cr.has_wrong_outputs THEN 'WRONG_OUTPUT'
      ELSE 'UNKNOWN'
    END AS error_classification,
    cr.has_api_failures,
    cr.has_wrong_actions,
    cr.has_wrong_outputs,
    cr.error_count
  FROM call_reviews cr
  JOIN soundflare_call_logs cl ON cr.call_log_id = cl.id
  JOIN soundflare_agents a ON cr.agent_id = a.id
  JOIN soundflare_projects p ON a.project_id = p.id
  WHERE cr.status = 'completed'
    AND cr.error_count > 0
    AND cl.created_at >= start_date
    AND cl.created_at < (end_date + INTERVAL '1 day')
    AND (
      error_type IS NULL
      OR (error_type = 'API_FAILURE' AND cr.has_api_failures = true)
      OR (error_type = 'WRONG_ACTION' AND cr.has_wrong_actions = true)
      OR (error_type = 'WRONG_OUTPUT' AND cr.has_wrong_outputs = true)
    )
  ORDER BY cl.call_started_at DESC;
$$;

REVOKE EXECUTE ON FUNCTION public.agency_failures(date, date, text) FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION public.agency_failures(date, date, text) FROM anon;
REVOKE EXECUTE ON FUNCTION public.agency_failures(date, date, text) FROM authenticated;
GRANT EXECUTE ON FUNCTION public.agency_failures(date, date, text) TO service_role;
```

### Slack Notification Tables
```sql
-- slack_notification_configs
CREATE TABLE IF NOT EXISTS public.slack_notification_configs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id uuid NOT NULL REFERENCES public.soundflare_agents(id) ON DELETE CASCADE,
  webhook_url text NOT NULL,
  is_enabled boolean NOT NULL DEFAULT true,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT slack_notification_configs_agent_id_key UNIQUE (agent_id)
);

ALTER TABLE public.slack_notification_configs ENABLE ROW LEVEL SECURITY;
-- No policies: only service_role (which bypasses RLS) can access

-- slack_notification_log
CREATE TABLE IF NOT EXISTS public.slack_notification_log (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  config_id uuid NOT NULL REFERENCES public.slack_notification_configs(id) ON DELETE CASCADE,
  agent_id uuid NOT NULL REFERENCES public.soundflare_agents(id) ON DELETE CASCADE,
  call_log_ids uuid[] NOT NULL,
  status varchar(20) NOT NULL DEFAULT 'sent' CHECK (status IN ('sent', 'failed')),
  error_message text,
  sent_at timestamptz NOT NULL DEFAULT now()
);

ALTER TABLE public.slack_notification_log ENABLE ROW LEVEL SECURITY;
-- No policies: only service_role can access

-- Indexes
CREATE INDEX idx_slack_notification_log_agent_id ON public.slack_notification_log(agent_id);
CREATE INDEX idx_slack_notification_log_sent_at ON public.slack_notification_log(sent_at);
CREATE INDEX idx_slack_notification_log_config_id ON public.slack_notification_log(config_id);

-- Grants: explicit for both tables
GRANT ALL ON public.slack_notification_configs TO service_role;
GRANT ALL ON public.slack_notification_log TO service_role;
-- No grants to anon or authenticated (RLS blocks + no grants = fully restricted)
```

### Index Strategy for Metrics Queries
```sql
-- Composite index for the metrics date-range + agent join
CREATE INDEX IF NOT EXISTS idx_call_logs_agent_created
  ON public.soundflare_call_logs(agent_id, created_at);

-- Index for call_reviews status filter (used in both RPCs)
-- idx_call_reviews_status already exists in schema
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Clerk auth (`@clerk/nextjs`) | Supabase GoTrue auth (`@supabase/ssr`) | Already migrated in codebase | `src/lib/auth.ts` wraps Supabase auth with Clerk-like API surface |
| Direct PostgREST queries from browser | Service-role client for sensitive data | Existing pattern | `getSupabaseAdmin()` in `src/lib/supabase-server.ts` |

**Key observation:** The codebase retains some Clerk-era naming conventions (e.g., `clerk_id` columns, `user_clerk_id` fields) but auth is fully Supabase/GoTrue now. The `auth()` and `currentUser()` functions in `src/lib/auth.ts` wrap Supabase auth calls.

## Open Questions

1. **PYPE_ADMINS value on production server**
   - What we know: The env var pattern exists in code (`notify-admins/route.ts`), format is comma-separated emails
   - What's unclear: Whether `PYPE_ADMINS` is already set in the production `.env.docker` on the DigitalOcean droplet
   - Recommendation: Ensure `PYPE_ADMINS` is in `.env.docker` before deploying. At minimum: `PYPE_ADMINS=admin@soundflare.ai`

2. **Agents without call logs**
   - What we know: Decision says "inactive agents included in results"
   - What's unclear: Whether the LEFT JOIN approach returns agents with NULL call_date (no calls in range) as a single row or zero rows
   - Recommendation: The LEFT JOIN will produce a single row with NULL call_date and zero counts for agents with no calls in range. Phase 2 UI handles display.

3. **PostgREST restart after migration**
   - What we know: PostgREST caches schema; new functions require restart
   - What's unclear: Whether there's an auto-reload mechanism configured
   - Recommendation: Include `docker compose restart rest kong` in post-migration steps

## Sources

### Primary (HIGH confidence)
- Codebase inspection: `database/schema-definitions.sql` -- full table schemas for all 23 tables
- Codebase inspection: `src/lib/supabase-server.ts` -- service-role client singleton
- Codebase inspection: `src/lib/auth.ts` -- auth helper wrapping Supabase getUser()
- Codebase inspection: `src/app/api/email/notify-admins/route.ts` -- existing PYPE_ADMINS parsing pattern
- Codebase inspection: `src/utils/supabase/middleware.ts` -- session refresh middleware
- Codebase inspection: `docker-compose.yml` -- PostgREST v10.1.1, GoTrue v2.94.0, PostgreSQL 15.1.1.2
- Codebase inspection: `database/setup-supabase.sql` -- role permissions (anon, authenticated, service_role)

### Secondary (MEDIUM confidence)
- PostgreSQL SECURITY DEFINER behavior -- standard PostgreSQL documentation

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already in project, no new dependencies
- Architecture: HIGH -- follows existing codebase patterns exactly
- Pitfalls: HIGH -- identified from direct codebase inspection (missing env var, PostgREST cache, permissions)

**Research date:** 2026-03-10
**Valid until:** 2026-04-10 (stable -- no fast-moving dependencies)
