# Soundflare — Agency Dashboard & Slack Alerts

## What This Is

Two new features for Soundflare: (1) a cross-project agency dashboard for the internal StrataBlue team to monitor agent performance across all projects — call counts, minutes, latency, failure rates, and a browsable failure list; (2) real-time Slack notifications when calls have failures detected by the existing AI call review system.

## Core Value

The internal team can see how every agent is performing at a glance and get immediately notified in Slack when something goes wrong, without having to check the app.

## Requirements

### Validated

- ✓ Call logs stored in `soundflare_call_logs` table — existing
- ✓ AI call reviews with error categorization (API_FAILURE, WRONG_ACTION, WRONG_OUTPUT) via `call_reviews` table — existing
- ✓ Per-project agent views and call log detail pages — existing
- ✓ CallReviewService processes reviews and stores results — existing

### Active

- [ ] Agency dashboard with cross-project aggregate metrics
- [ ] Failure list with drill-down to existing call detail pages
- [ ] Per-agent Slack channel configuration for failure notifications
- [ ] Real-time Slack alerts on review-detected failures

### Out of Scope

- Client-facing dashboard — internal team only for now
- Batched/digest Slack notifications — real-time only
- Custom date range picker — preset ranges (today, 7d, 30d, 90d) only
- Full Slack OAuth app install — using existing Slack app with webhook URLs
- New failure detection beyond existing call review categories
- Mobile/responsive optimization of dashboard

## Context

Soundflare is a voice AI agent platform built with Next.js 15, Supabase (PostgreSQL), and integrations with Trillet, OpenAI, Google Gemini, and others. It runs on a DigitalOcean droplet via Docker Compose.

The existing call review system (CallReviewService) already analyzes calls using Google Gemini and categorizes failures as API_FAILURE, WRONG_ACTION, or WRONG_OUTPUT. These reviews are stored in the `call_reviews` table. The agency dashboard aggregates data from `soundflare_call_logs` and `call_reviews` across all projects.

For Slack, the team has an existing Slack app that can be reused. Each agent will have its own Slack webhook URL configured, allowing failures to route to agent-specific channels.

The app currently uses a project-scoped URL structure (`/[projectid]/...`). The agency dashboard will be a new top-level route outside of any specific project.

## Constraints

- **Stack**: Must use existing Next.js 15 + Supabase stack, Radix UI components, TanStack React Query, Recharts for charts, Tailwind CSS
- **Data**: All metrics derived from existing `soundflare_call_logs` and `call_reviews` tables — no new data ingestion
- **Auth**: Internal team only — use existing Supabase auth, restricted to admin users (PYPE_ADMINS list)
- **Deployment**: Same Docker Compose setup on DigitalOcean droplet
- **Slack**: Webhook-based integration using existing Slack app — no OAuth flow needed

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Cross-project dashboard (not per-project) | Team needs aggregate view across all clients/agents | — Pending |
| Failures = call review categories | Reuse existing AI review system rather than defining new failure criteria | — Pending |
| Per-agent Slack channels | Different agents may have different teams/stakeholders monitoring them | — Pending |
| Webhook URL config (not Slack OAuth) | Simpler integration, existing Slack app available | — Pending |
| Preset time ranges only | Keeps UI simple, covers primary use cases | — Pending |

---
*Last updated: 2026-03-10 after initialization*
