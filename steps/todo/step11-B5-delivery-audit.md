# Step 11-B5 — Delivery Audit and Monitoring (11.9)

## Problem

The team has no way to see delivery quality across calls. Individual call records exist in `calls` and `audit_logs`, and per-device ack data exists in `call_delivery_acks` and `call_delivery_attempts`, but there's no aggregated view. Delivery regressions (e.g., a broken APNs cert, a new Android OS version suppressing notifications) go unnoticed until residents complain.

## Dependencies

- **B2** (persist ringing sessions) — `call_delivery_attempts` table must exist
- **Phase A** (device acks, callId) — `call_delivery_acks` table must exist

## What To Do

### Delivery summary view

Create a SQL view that aggregates delivery data per call.

Migration `008-delivery-summary-view.sql`:

```sql
CREATE OR REPLACE VIEW call_delivery_summary AS
SELECT
    c.id AS call_id,
    c.building_id,
    c.apartment_id,
    c.intercom_id,
    c.status AS call_status,
    c.created_at AS call_started_at,
    c.ended_at AS call_ended_at,
    COUNT(DISTINCT cda.device_token) AS devices_targeted,
    COUNT(DISTINCT cda.device_token) FILTER (WHERE cda.delivery_state IN ('push-received', 'app-awake', 'incoming-ui-shown', 'accepted')) AS devices_acked,
    COUNT(DISTINCT cda.device_token) FILTER (WHERE cda.delivery_state = 'push-failed') AS devices_failed,
    COUNT(DISTINCT cda.device_token) FILTER (WHERE cda.delivery_state = 'timed-out') AS devices_timed_out,
    COUNT(DISTINCT cda.device_token) FILTER (WHERE cda.delivery_state = 'skipped-sleep-mode') AS devices_skipped_sleep,
    MAX(cda.attempt_number) AS max_retries,
    MIN(cdacks.created_at) FILTER (WHERE cdacks.event = 'push-received') AS first_ack_at,
    EXTRACT(EPOCH FROM (MIN(cdacks.created_at) FILTER (WHERE cdacks.event = 'push-received') - c.created_at)) AS first_ack_latency_sec
FROM calls c
LEFT JOIN call_delivery_attempts cda ON cda.call_id = c.id
LEFT JOIN call_delivery_acks cdacks ON cdacks.call_id = c.id
GROUP BY c.id;
```

### Management portal additions

Add a "Delivery Health" section to the management portal that shows:

1. **Recent calls** with delivery outcome (answered / unanswered / rejected) and device-level breakdown
2. **Delivery rate** over the last 7 days: % of calls where at least one device acked
3. **Average ack latency** by platform (iOS vs Android)
4. **Failed deliveries** grouped by error type
5. **Apartments with no healthy tokens** (all tokens stale or no tokens registered)

This can be a simple read-only page querying the view above, or API endpoints consumed by the existing management portal.

### Admin portal additions

Add a "System Delivery Health" section to the admin portal showing cross-building metrics:

1. Delivery rate by building
2. Token health: total tokens, stale tokens pruned in the last 7 days
3. Calls with `delivery-degraded` audit events
4. Retry effectiveness: % of retried devices that eventually acked

### SQL migration

Create `008-delivery-summary-view.sql` with the view definition above.

## Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/sql/008-delivery-summary-view.sql` | New migration — delivery summary view |
| `vidrom-signaling-server/lambda/managementRoutes.js` | Add delivery health API endpoints |
| `vidrom-signaling-server/lambda/adminRoutes.js` | Add system-wide delivery health endpoints |
| `vidrom-signaling-server/portals/management.html` | Delivery health UI section |
| `vidrom-signaling-server/portals/admin.html` | System delivery health UI section |

## Verification

- [ ] `call_delivery_summary` view returns correct aggregated data
- [ ] Management portal shows recent calls with device-level delivery breakdown
- [ ] Admin portal shows cross-building delivery rate and token health
- [ ] `delivery-degraded` calls are surfaced prominently
- [ ] Queries perform acceptably on production data volumes
