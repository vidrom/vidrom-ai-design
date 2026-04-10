# Step 11-B3 — Retry Orchestration for Unacknowledged Devices (11.4)

## Problem

Phase A sends push notifications once and hopes for the best. If a push is silently dropped by the OS, or a device is slow to wake, there is no retry. The server now has ack data (Phase A) and durable state (B2), so it can identify unacknowledged devices and retry.

## Dependencies

- **B2** (persist ringing sessions) — retry state must be durable
- **Phase A** (device acks) — retries are driven by missing acks

## What To Do

### Retry timeline

After the initial ring push at `t=0`:

| Time | Action |
|------|--------|
| `t=0s` | Send WS ring + push to all eligible devices. Insert `call_delivery_attempts` rows. |
| `t=3s` | **Retry 1**: For devices with no ack (`delivery_state` still `push-sent`), resend push. Update `attempt_number` to 2. |
| `t=8s` | **Retry 2**: For devices still unacked, resend push. Update `attempt_number` to 3. Log a warning per apartment. |
| `t=15s` | **Final check**: If zero devices have acked, log `delivery-degraded` for this call. (Escalation to fallback channels is Phase C.) |

The retry window must stay within the building's `no_answer_timeout`. If timeout is 30s, retries at 3s, 8s, and 15s are safe. If timeout is shorter (e.g., 20s), skip the 15s retry.

### Server — retry orchestrator

Add a retry scheduler in a new module `retryOrchestrator.js`:

1. When a ring starts, schedule retry checks at the defined intervals using `setTimeout`.
2. At each retry tick:
   - Query `call_delivery_attempts` for this `call_id` where `delivery_state = 'push-sent'` (never acked).
   - For each unacked device, resend the same push (FCM or VoIP).
   - Insert a new `call_delivery_attempts` row with incremented `attempt_number`.
3. Cancel all pending retries when:
   - The call is accepted (any device)
   - The call is declined (all residents)
   - The ring expires (timeout)
   - The intercom hangs up

### Idempotency

The home app already handles duplicate ring pushes gracefully (it checks if it's already showing the incoming call UI). Retried pushes carry the same `callId`, so the app can deduplicate.

### Delivery-degraded logging

If no device acks by the final retry check, insert an `audit_logs` entry:
```
event_type: 'delivery-degraded'
description: 'No device acknowledged ring for call <callId> after 3 attempts'
```

This feeds into the monitoring dashboard (B5).

## Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/retryOrchestrator.js` | New module — retry scheduling and execution |
| `vidrom-signaling-server/src/wsHandler.js` | Call retry orchestrator on ring start; cancel on accept/decline/hangup/timeout |
| `vidrom-signaling-server/src/connectionState.js` | Expose retry cancellation on call clear |

## Verification

- [ ] Devices that ack within 3s are not retried
- [ ] Unacked devices get a retry push at ~3s and ~8s
- [ ] Retry inserts new `call_delivery_attempts` rows with incremented `attempt_number`
- [ ] Retries stop immediately when call is accepted, declined, or hung up
- [ ] `delivery-degraded` audit log fires when zero devices ack after all retries
- [ ] Duplicate ring pushes are handled gracefully by the home app
- [ ] Retry intervals respect the building's `no_answer_timeout`
