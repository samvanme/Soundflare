# Roadmap: Soundflare Agency Dashboard & Slack Alerts

## Overview

This milestone adds two capabilities to the existing Soundflare application: a cross-project agency dashboard for monitoring agent performance metrics, and Slack failure notifications triggered by the existing AI call review system. The build order follows data dependencies: database objects and auth guards first, then the dashboard UI that consumes them, then the Slack notification pipeline that modifies the existing review service.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Database & Auth Foundation** - SQL views, RPC functions, new tables, indexes, and admin auth guards
- [ ] **Phase 2: Agency Dashboard** - Cross-project metrics UI with KPI cards, agent table, failure list, trend charts, and health indicators
- [ ] **Phase 3: Slack Notifications** - Webhook-based failure alerting with per-agent configuration, throttling, and integration into CallReviewService

## Phase Details

### Phase 1: Database & Auth Foundation
**Goal**: All database objects and auth guards exist so that dashboard and notification features can be built on a verified data layer
**Depends on**: Nothing (first phase)
**Requirements**: INFR-01, INFR-02, INFR-03
**Success Criteria** (what must be TRUE):
  1. Admin user accessing `/agency` is authenticated and authorized; non-admin authenticated users are rejected
  2. Calling the agency metrics RPC function returns aggregated call counts, minutes, and failure rates grouped by agent and date, computed entirely in PostgreSQL
  3. Calling the agency failures RPC function returns failed call records with error type classification, supporting date range and error type filter parameters
  4. `slack_notification_configs` and `slack_notification_log` tables exist with appropriate columns and admin-only access (no browser client access)
**Plans**: 2 plans

Plans:
- [ ] 01-01-PLAN.md — SQL migrations: agency RPC functions, Slack notification tables, indexes, and permission grants
- [ ] 01-02-PLAN.md — Admin auth guard layout, assertAgencyAdmin helper, and agency API routes (metrics + failures)

### Phase 2: Agency Dashboard
**Goal**: Admin users can view cross-project agent performance metrics, spot degradation trends, and drill into specific failures from a single page
**Depends on**: Phase 1
**Requirements**: DASH-01, DASH-02, DASH-03, DASH-04, DASH-05, DASH-06, FAIL-01, FAIL-02, FAIL-03
**Success Criteria** (what must be TRUE):
  1. Admin user can view summary cards showing total calls, total minutes, average failure rate, and active agent count across all projects
  2. Admin user can view a per-agent table with call count, failure count, failure rate, last call timestamp, health indicator, and failure rate sparkline
  3. Admin user can select a preset time range (today, 7d, 30d, 90d) and all dashboard data updates accordingly
  4. Admin user can view daily call volume and failure rate trend charts, plus an error type distribution chart
  5. Admin user can view a filterable failure list and click any failed call to navigate to its existing call detail page
**Plans**: TBD

Plans:
- [ ] 02-01: TBD
- [ ] 02-02: TBD

### Phase 3: Slack Notifications
**Goal**: The team receives Slack alerts when agent calls fail, with per-agent channel routing, throttling to prevent alert fatigue, and a configuration UI for managing webhooks
**Depends on**: Phase 1
**Requirements**: SLCK-01, SLCK-02, SLCK-03, SLCK-04
**Success Criteria** (what must be TRUE):
  1. Admin user can configure a Slack webhook URL for any agent and test it with a confirmation message
  2. When a call review detects failures, a Slack notification is sent to the configured agent channel with agent name, error types, and a direct link to the call
  3. Rapid consecutive failures for the same agent are batched into a combined alert instead of individual notifications
  4. Webhook URLs are never exposed to the browser client; the configuration UI displays masked URLs
**Plans**: TBD

Plans:
- [ ] 03-01: TBD
- [ ] 03-02: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Database & Auth Foundation | 0/2 | Planned | - |
| 2. Agency Dashboard | 0/0 | Not started | - |
| 3. Slack Notifications | 0/0 | Not started | - |
