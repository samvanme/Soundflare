# Phase 1: Database & Auth Foundation - Context

**Gathered:** 2026-03-10
**Status:** Ready for planning

<domain>
## Phase Boundary

All database objects (RPC functions, tables, indexes) and auth guards so that the agency dashboard (Phase 2) and Slack notifications (Phase 3) can be built on a verified data layer. This phase delivers no UI beyond the auth guard redirects.

**Terminology note:** Soundflare uses "organizations" (not "projects") as the user-facing term for the entities in `soundflare_projects`. The agency dashboard aggregates across all organizations for internal StrataBlue ops.

</domain>

<decisions>
## Implementation Decisions

### Admin identity model
- Agency admins identified via `PYPE_ADMINS` environment variable — comma-separated email allowlist
- No new database table for admin identity; env var is sufficient for internal team
- Non-admin authenticated users hitting `/agency` are silently redirected to home (`/`)
- Unauthenticated users hitting `/agency` are redirected to `/sign-in?next=/agency` (return URL preserved)

### Metrics data scope
- Agency metrics aggregate across ALL organizations — no filtering or exclusion
- Each agent row includes its organization name (from `soundflare_projects.name`)
- Date granularity is daily (one row per agent per day) — feeds Phase 2 trend charts
- Inactive agents (no recent calls) are included in results — Phase 2 handles display

### Notification table design
- One webhook URL per agent (`UNIQUE(agent_id)` constraint on `slack_notification_configs`)
- `is_enabled` boolean toggle per config
- Throttle window is hardcoded in code (not a per-agent column) — sensible default, adjust in code if needed
- `slack_notification_log` tracks: config_id, agent_id, call_log_ids (UUID array), status ('sent'/'failed'), error_message, sent_at
- Both tables CASCADE delete when parent agent is deleted (consistent with existing call_logs/call_reviews pattern)

### Auth guard pattern
- Layout-level check in `/agency/layout.tsx` (server component) — checks PYPE_ADMINS after verifying authentication
- API routes under `/api/agency/*` also check PYPE_ADMINS via shared `assertAgencyAdmin()` helper — prevents direct API bypass
- Middleware stays as-is (session refresh only) — no admin logic in middleware
- RLS on `slack_notification_configs` and `slack_notification_log`: enabled with no anon/authenticated policies (service role only)
- RPC functions (`agency_metrics`, `agency_failures`): SECURITY DEFINER with EXECUTE revoked from public/anon — callable only via service role from API routes

### Claude's Discretion
- Exact RPC function signatures and return types
- Index strategy for metrics queries
- Throttle window duration (suggested ~5 minutes)
- Error type classification logic within the failures RPC
- SQL migration file organization

</decisions>

<specifics>
## Specific Ideas

- Auth pattern: layout.tsx handles auth + authorization, shared helper for API routes
- Service role pattern: all agency data flows through server-side API routes, never exposed to browser client
- Consistent with existing codebase patterns: CASCADE deletes, RLS policies, PostgREST/Supabase conventions

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-database-auth-foundation*
*Context gathered: 2026-03-10*
