# Requirements: Soundflare Agency Dashboard & Slack Alerts

**Defined:** 2026-03-10
**Core Value:** The internal team can see how every agent is performing at a glance and get immediately notified in Slack when something goes wrong

## v1 Requirements

### Dashboard

- [ ] **DASH-01**: Admin user can view cross-project summary cards showing total calls, total minutes, average failure rate, and active agent count
- [ ] **DASH-02**: Admin user can view a per-agent metrics table with call count, failure count, failure rate, and last call timestamp
- [ ] **DASH-03**: Admin user can filter all dashboard data by preset time range (today, 7d, 30d, 90d)
- [ ] **DASH-04**: Admin user can view daily call volume and failure rate trend charts over the selected time range
- [ ] **DASH-05**: Admin user can see a health status indicator (green/yellow/red) per agent based on recent failure rate
- [ ] **DASH-06**: Admin user can view an error type distribution chart (API_FAILURE vs WRONG_ACTION vs WRONG_OUTPUT)

### Failures

- [ ] **FAIL-01**: Admin user can view a list of calls with review-detected failures, filterable by error type
- [ ] **FAIL-02**: Admin user can click a failed call to navigate to the existing call detail/observability page
- [ ] **FAIL-03**: Admin user can see a failure rate sparkline trend per agent in the metrics table

### Slack

- [ ] **SLCK-01**: Admin user can configure a Slack webhook URL per agent
- [ ] **SLCK-02**: System sends a Slack notification when a call review detects failures, including agent name, error types, and a direct link to the call
- [ ] **SLCK-03**: Admin user can test a configured webhook URL and see delivery confirmation
- [ ] **SLCK-04**: System batches rapid consecutive failures for the same agent into a combined alert instead of sending individual notifications

### Infrastructure

- [ ] **INFR-01**: Dashboard route is restricted to admin users (PYPE_ADMINS list)
- [ ] **INFR-02**: Dashboard metrics are computed via PostgreSQL RPC functions, not client-side aggregation
- [ ] **INFR-03**: Slack webhook URLs are stored securely and accessed only through server-side API routes

## v2 Requirements

### Dashboard Enhancements

- **DASH-07**: Client-facing dashboard with per-project scoped view
- **DASH-08**: Custom date range picker with arbitrary start/end dates
- **DASH-09**: Real-time live-updating dashboard via WebSocket/SSE

### Alerting Enhancements

- **SLCK-05**: Custom alerting rules and threshold configuration per agent
- **SLCK-06**: Multi-channel alerting (email, PagerDuty) beyond Slack
- **SLCK-07**: Daily/weekly digest summaries in addition to real-time alerts

## Out of Scope

| Feature | Reason |
|---------|--------|
| Full Slack OAuth app installation | Using existing Slack app with webhook URLs — OAuth adds massive complexity for an internal tool |
| Mobile/responsive dashboard optimization | Internal ops tool used on desktop only |
| New failure detection categories | Reusing existing call review categories (API_FAILURE, WRONG_ACTION, WRONG_OUTPUT) |
| Custom alert threshold configuration UI | Hardcode sensible defaults; adjust in code if needed |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| DASH-01 | Phase 2 | Pending |
| DASH-02 | Phase 2 | Pending |
| DASH-03 | Phase 2 | Pending |
| DASH-04 | Phase 2 | Pending |
| DASH-05 | Phase 2 | Pending |
| DASH-06 | Phase 2 | Pending |
| FAIL-01 | Phase 2 | Pending |
| FAIL-02 | Phase 2 | Pending |
| FAIL-03 | Phase 2 | Pending |
| SLCK-01 | Phase 3 | Pending |
| SLCK-02 | Phase 3 | Pending |
| SLCK-03 | Phase 3 | Pending |
| SLCK-04 | Phase 3 | Pending |
| INFR-01 | Phase 1 | Pending |
| INFR-02 | Phase 1 | Pending |
| INFR-03 | Phase 1 | Pending |

**Coverage:**
- v1 requirements: 16 total
- Mapped to phases: 16
- Unmapped: 0

---
*Requirements defined: 2026-03-10*
*Last updated: 2026-03-10 after roadmap creation*
