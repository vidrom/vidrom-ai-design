# Step 11 — Production Call Delivery Reliability

## Goal

Strengthen the intercom-to-home call delivery pipeline so residents are extremely unlikely to miss an incoming intercom call, even when the phone is locked, the app is backgrounded, the device is slow to wake, or one delivery channel fails.

## Overview

The current system already uses three strong mechanisms:

- WebSocket ring delivery to connected home apps
- FCM push delivery for Android and fallback delivery
- APNs VoIP push for iOS CallKit wake-up

That is a good baseline, but in production it is still a best-effort fanout model. To reduce missed calls further, the system should evolve into a delivery workflow with explicit per-device acknowledgements, durable call state, retries, and escalation paths.

This step proposes that production reliability layer.

## Problems To Solve

The resident can still miss a call in cases such as:

- push sent but not actually displayed by the OS
- app wakes too slowly and the ring expires before reconnect
- one device is offline while another has stale tokens
- server restarts or transient infra failures interrupt a pending ring
- user taps answer from a locked screen but the app needs too long to fully reconnect
- all push channels fail for a specific apartment and there is no escalation path

## Design Principles

1. Treat call delivery as a tracked workflow, not a fire-and-forget notification.
2. Track delivery separately per device, not only per apartment.
3. Persist active ringing state so retries survive process restarts.
4. Allow a resident to reserve the call before full app reconnection finishes.
5. Detect unhealthy devices early and surface that risk.
6. Provide a fallback path when normal push delivery fails.

---

## Step 11.1 — Add a Call ID and Delivery State Machine

Every intercom ring should create a unique `callId` on the signaling server.

The `callId` should be included in:

- WebSocket `ring`
- FCM `incoming-call`
- APNs VoIP `incoming-call`
- `accept`, `decline`, `hangup`, and `call-taken`

Track delivery state per device, not only per apartment. Suggested device states:

- `queued`
- `push-sent`
- `push-received`
- `app-awake`
- `incoming-ui-shown`
- `accepted`
- `declined`
- `timed-out`
- `delivery-failed`

This allows the backend to know whether a device was truly alerted, rather than only knowing that a push request was issued.

## Step 11.2 — Add Device Acknowledgement Events

Each home app should report lightweight delivery acknowledgements back to the server for the current `callId`.

Suggested acknowledgements:

- push received in background handler
- app resumed or launched because of this call
- CallKit or full-screen incoming UI displayed
- user pressed answer
- user pressed decline

These can be delivered over HTTP or WebSocket. HTTP is useful because the app may not yet have a fully ready signaling socket.

This creates positive confirmation that the device was actually reached.

## Step 11.3 — Persist Active Ringing Sessions In The Database

The current in-memory pending ring should be replaced or backed by durable storage.

Suggested tables:

- `call_sessions`
- `call_delivery_attempts`

`call_sessions` should store:

- `call_id`
- `intercom_device_id`
- `building_id`
- `apartment_id`
- `status`
- `created_at`
- `expires_at`
- `accepted_by_user_id`
- `accepted_by_device_token`

`call_delivery_attempts` should store:

- `call_id`
- `user_id`
- `device_token`
- `token_type`
- `platform`
- `attempt_number`
- `delivery_state`
- `last_error`
- `last_attempt_at`
- `acked_at`

This ensures active ringing and retry state survive a signaling server restart.

## Step 11.4 — Add Retry Orchestration For Unacknowledged Devices

If a device does not acknowledge receipt within a short time window, retry delivery.

Suggested staged timeline:

1. At `t=0s`, send WS ring and push notifications to all eligible devices.
2. At `t=2s`, retry push for devices with no acknowledgement.
3. At `t=6s`, retry again and mark the apartment as degraded if still no device acked.
4. At `t=10s`, escalate to fallback channels if configured.

Retries should be tracked by the server, not by ad hoc logs. They should be idempotent and tied to the `callId`.

## Step 11.5 — Add Direct Accept Over HTTP

Residents should be able to reserve the call immediately from the lock-screen flow without waiting for the full WebSocket and WebRTC stack to come up.

Add a signed HTTP endpoint such as:

- `POST /api/home/calls/:callId/accept`

Behavior:

1. Home app receives push for `callId`
2. User taps answer from CallKit or notification UI
3. App immediately sends HTTP accept for `callId`
4. Server marks the call as reserved by that resident device
5. App then establishes WS and WebRTC normally

This reduces answer races caused by slow reconnects on locked or cold-started devices.

## Step 11.6 — Extend Ring Lifetime And Allow Late Join

The active ring should remain recoverable long enough for slow devices to wake and reconnect.

Recommended changes:

- increase ring lifetime to a more forgiving production value
- allow a recently awakened device to resume a still-ringing `callId`
- if the call is already accepted elsewhere, return an explicit `call-taken` outcome immediately

This prevents a user from seeing the incoming call UI but losing the race only because the server expired the ring too early.

## Step 11.7 — Add Device Health Tracking

Track whether each resident device is currently healthy enough to receive urgent calls.

Suggested health signals:

- last successful push delivery
- last token refresh time
- app version
- OS version
- notification permission state
- battery optimization risk on Android
- last foreground time
- last successful incoming call acknowledgement

For each apartment, compute whether there is at least one healthy device.

If an apartment has no healthy devices, surface that in admin or management tools before a real visitor call happens.

## Step 11.8 — Add Fallback Escalation Paths

If no device in the apartment acknowledges a ring within the defined time window, the system should optionally escalate.

Possible fallback channels:

- SMS with deep link into the app
- automated phone call to the resident
- alternate resident contact methods
- concierge or building-level fallback workflow

This is the last line of defense for residents whose phones are offline, misconfigured, or blocked by OS-level restrictions.

## Step 11.9 — Add Delivery Audit And Monitoring

Add operational visibility for call delivery.

The system should record:

- which devices were targeted for each `callId`
- which channel was used for each attempt
- whether the device acknowledged receipt
- whether a resident answered or declined
- delivery latency by platform and token type
- failures by apartment, building, app version, and device model

Provide a simple dashboard or queryable report so the team can detect delivery regressions before residents report them.

## Step 11.10 — iOS Critical Alerts Entitlement

Apple provides a Critical Alerts entitlement that allows notifications to bypass Do Not Disturb, Focus modes, and silent mode. This is designed for health, safety, and security apps — an intercom system is a textbook qualifying use case.

Without this entitlement, a resident who enables Focus mode ("Sleep", "Work", "Driving") on their iPhone will silently miss every intercom call, and no amount of retry or escalation logic can help because the OS itself suppresses delivery.

Steps:

- Apply for the Critical Alerts entitlement via Apple Developer → Request entitlement form
- Add the `com.apple.developer.usernotifications.critical-alerts` entitlement to the home app
- When sending FCM or local notifications on iOS, set `sound: { critical: true, volume: 1.0 }` in the APNs payload
- VoIP pushes via PushKit already bypass DND, but if the user's CallKit is suppressed by Focus, Critical Alerts on the FCM notification path act as a secondary wakeup

This is one of the highest-leverage additions because it addresses a failure mode no amount of server-side logic can fix.

## Step 11.11 — Intercom-Side Visitor Feedback During Ringing

The intercom InCallScreen currently shows only "Ringing..." with no information about delivery status. The visitor standing at the door has no idea whether anyone is being reached.

The server should push real-time delivery progress back to the intercom's WebSocket:

- `{ type: 'ring-progress', devicesNotified: 3 }` — after push fanout completes
- `{ type: 'ring-progress', devicesConfirmed: 1 }` — when at least one device acknowledges
- `{ type: 'ring-progress', noResponse: true }` — if no device acknowledges within the retry window

The intercom UI should display this to the visitor:

- "Ringing apartment 5... (3 devices notified)"
- "Ringing apartment 5... (1 device confirmed)"
- "No response from apartment 5 — try again or use door code"

This dramatically improves visitor experience and helps building managers diagnose problems on site.

## Step 11.12 — Respect Per-User Sleep Mode In Delivery Workflow

The business specs define a `sleep_mode` boolean per user. The delivery workflow should:

- Skip push delivery to devices belonging to users with `sleep_mode = true`
- Still track them in the delivery state machine as `skipped-sleep-mode`
- If ALL residents of the target apartment have sleep mode enabled, do not ring at all and immediately inform the intercom: `{ type: 'apartment-unavailable', reason: 'all-residents-sleeping' }`
- The intercom should display "Residents are currently unavailable" instead of ringing into the void

This prevents wasted delivery attempts and tells the visitor the truth.

## Step 11.13 — Use Building-Level No-Answer Timeout

The database already has `no_answer_timeout` per building (default 30 seconds). The server currently uses a hardcoded `30000` ms in `connectionState.js`.

The ring lifetime and pending ring timeout should be driven by the building's setting, not a global constant. When an intercom rings:

- Query the building's `no_answer_timeout`
- Use that value for the pending ring expiry timer
- Use that value for the retry orchestration window
- If the building setting is not available, fall back to the global setting in `global_settings`

This allows building managers to tune ring behavior per building without code changes.

## Step 11.14 — Reactive Stale Token Cleanup

When a push attempt returns a known-bad-token error, immediately prune that token from `device_tokens`:

- FCM: `messaging/registration-token-not-registered`, `messaging/invalid-registration-token`
- APNs: `BadDeviceToken`, `Unregistered`

Do not wait for periodic health scoring. Reactive cleanup prevents the same dead token from silently absorbing delivery attempts on every subsequent ring.

Also clean tokens when:

- A user explicitly logs out of the home app
- The app detects a fresh install and sends a new token (delete old tokens for the same user)

## Step 11.15 — WebSocket Ping/Pong Heartbeat For Intercom Connections

The intercom maintains a persistent WebSocket, but neither side sends periodic pings. A zombie TCP connection can appear open for minutes before the OS raises an error.

During that window, the server believes the intercom is connected, but any ring sent over the dead socket is silently lost — and since the ring originates from the intercom, a zombie connection means the ring never even reaches the server.

Add server-side WebSocket ping every 15–30 seconds. If the intercom does not respond with a pong within 10 seconds, close the socket and update `intercoms.status` to `disconnected`.

On the intercom side, detect missing pings and trigger reconnection proactively rather than waiting for the OS-level close event.

## Step 11.16 — Integrate With Existing Calls And Audit Log Tables

The database already has a `calls` table (with statuses `calling`, `accepted`, `ended`, `rejected`, `unanswered`) and an `audit_logs` table, but the signaling server does not currently write to either.

The new `call_sessions` and `call_delivery_attempts` tables proposed in Step 11.3 should complement, not replace, the existing `calls` table:

- When a ring starts, insert a row into `calls` with status `calling` and use its `id` as the `callId`
- When the call is accepted, update `calls.status` to `accepted`
- When the call ends, update `calls.status` and `ended_at`
- If the ring expires with no answer, update to `unanswered`
- Write corresponding `audit_logs` entries for `call-initiated`, `call-accepted`, `call-rejected`, `call-ended`, `call-unanswered`

`call_delivery_attempts` is an operational detail table that tracks per-device delivery mechanics. `calls` and `audit_logs` remain the source of truth for call history visible to management and admin portals.

## Step 11.17 — Support Missed-Call Recovery

If a device wakes too late, the app should still know what happened.

Recommended behavior:

- if the call is still ringing, offer immediate join
- if another resident already answered, show that explicitly
- if the ring expired, show a missed-call event with timestamp and building/intercom context

This reduces ambiguity for the resident and gives support a clear audit trail.

---

## Priorities And Recommended Order

The table below ranks every sub-step by impact, effort, and risk. Items near the top should be done first — they either fix the most likely real-world failure modes, require the least effort, or unblock later items.

| Priority | Step | Name | Impact | Effort | Why this order |
|----------|------|------|--------|--------|----------------|
| **1** | 11.15 | WS ping/pong heartbeat | High | Small | A zombie intercom socket means the ring never reaches the server at all. Fixing this is a few dozen lines on the server and intercom, and it eliminates an invisible failure mode. |
| **2** | 11.14 | Reactive stale token cleanup | High | Small | Dead tokens silently absorb every delivery attempt. Pruning on known-bad errors is a small change in the ring handler's `.catch()` blocks with immediate payoff. |
| **3** | 11.13 | Building no-answer timeout | Medium | Small | Replace one hardcoded `30000` with a DB query. Tiny change, but it makes ring lifetime configurable per building and unblocks retry tuning. |
| **4** | 11.1 | Call ID and delivery state machine | High | Medium | Foundation for everything else — retries, acks, audit, and HTTP accept all depend on a `callId`. Do this before any other medium-effort item. |
| **5** | 11.16 | Integrate with calls + audit_logs | High | Medium | Use the `callId` from 11.1 to write to the existing `calls` and `audit_logs` tables. This gives management/admin visibility and is a natural extension of 11.1. |
| **6** | 11.2 | Device acknowledgement events | High | Medium | With `callId` in place, add lightweight ack HTTP endpoints. Required before retry orchestration (11.4) can know which devices to retry. |
| **7** | 11.5 | Direct HTTP accept | High | Medium | Lets a locked/cold-started device reserve the call instantly. Depends on `callId` (11.1). High value because it eliminates the reconnect race. |
| **8** | 11.12 | Sleep mode in delivery workflow | Medium | Small | Query `sleep_mode` before sending push. Small server change, but prevents wasted delivery and tells the visitor the truth. |
| **9** | 11.6 | Extend ring lifetime / late join | Medium | Small | With building timeout (11.13) and callId (11.1) in place, this is a small logic adjustment to allow recently-awakened devices to rejoin a still-ringing call. |
| **10** | 11.3 | Persist ringing sessions in DB | High | Medium | Move pending ring from in-memory to durable storage. Needed before retry orchestration can survive server restarts. |
| **11** | 11.4 | Retry orchestration | High | Large | The staged retry timer (2s / 6s / 10s). Depends on acks (11.2) and durable state (11.3). High complexity but high payoff. |
| **12** | 11.11 | Intercom visitor feedback | Medium | Medium | Push `ring-progress` events to the intercom. Depends on acks (11.2) flowing into the server. |
| **13** | 11.9 | Delivery audit and monitoring | Medium | Medium | Tables and queries for delivery quality. Lower urgency but valuable for diagnosing regressions once the delivery pipeline is instrumented. |
| **14** | 11.17 | Missed-call recovery | Medium | Medium | Show the resident what happened. Depends on callId (11.1) and call history in DB (11.16). |
| **15** | 11.7 | Device health tracking | Medium | Large | Aggregate health signals per device. Valuable for proactive alerting but depends on acks and audit data accumulating first. |
| **16** | 11.10 | iOS Critical Alerts | High | External | Requires Apple entitlement approval (weeks). Start the application early, but actual integration can happen whenever the entitlement is granted. |
| **17** | 11.8 | Fallback escalation (SMS, phone) | Medium | Large | Last line of defense. Largest scope (third-party SMS/voice APIs, per-resident config). Do this after the core delivery pipeline is solid. |

### Dependency graph (simplified)

```
11.15 (heartbeat)          — standalone, do first
11.14 (stale tokens)       — standalone, do first
11.13 (building timeout)   — standalone, do first
        ↓
11.1  (callId)             — foundation
        ↓
   ┌────┴──────────────┐
11.16 (calls + audit)  11.2 (acks)  11.5 (HTTP accept)  11.12 (sleep mode)
        ↓                 ↓
11.17 (missed-call)    11.3 (persist ringing)
                          ↓
                       11.4 (retry orchestration)
                          ↓
                  ┌───────┴────────┐
               11.11 (visitor UX)  11.9 (audit dashboard)
                                      ↓
                                   11.7 (health scoring)
                                      ↓
                                   11.8 (fallback escalation)

11.10 (Critical Alerts)  — external dependency, submit early, integrate when granted
11.6  (late join)         — after 11.1 + 11.13
```

## Suggested Rollout Plan

### Phase A — Immediate Reliability Wins

- add `callId` and integrate with existing `calls` table (11.1, 11.16)
- add per-device acknowledgements (11.2)
- add direct HTTP accept (11.5)
- use building-level `no_answer_timeout` instead of hardcoded 30s (11.13)
- add reactive stale token cleanup on known-bad push errors (11.14)
- add WebSocket ping/pong heartbeat for intercom connections (11.15)

### Phase B — Durable Delivery Workflow

- persist ringing sessions in DB (11.3)
- add retry orchestration for unacknowledged devices (11.4)
- add delivery audit tables and reports (11.9)
- respect per-user sleep mode in delivery fanout (11.12)
- push delivery progress to intercom for visitor feedback (11.11)

### Phase C — Production Hardening

- apply for and integrate iOS Critical Alerts entitlement (11.10)
- add device health scoring (11.7)
- add fallback escalation channels (11.8)
- add late-join and missed-call recovery UX (11.6, 11.17)

## Success Criteria

This step is complete when:

1. Every intercom ring has a durable `callId` tracked end-to-end, written to the existing `calls` and `audit_logs` tables.
2. The server knows which specific devices actually received and displayed the call.
3. A resident can reserve the call before full signaling reconnect completes.
4. Ring delivery survives process restarts and transient push failures.
5. The team can inspect delivery quality per apartment and per device.
6. The system has a defined fallback path when ordinary push delivery fails.
7. iOS residents with Focus/DND enabled still receive incoming call alerts via Critical Alerts or VoIP push.
8. The intercom visitor sees real-time delivery feedback instead of a blind "Ringing..." screen.
9. Sleep mode is respected per-resident, and the visitor is informed when all residents are unavailable.
10. Ring timeout is driven by the building's `no_answer_timeout` setting, not a hardcoded constant.
11. Stale push tokens are pruned reactively on delivery failure, not only during periodic health checks.
12. Zombie intercom WebSocket connections are detected and closed within seconds via ping/pong heartbeat.

## Notes

- This step is intentionally focused on delivery reliability, not media quality.
- TURN/STUN reliability is covered separately by `step11-stun-turn-server-coturn.md` in `steps/done`.
- The mechanisms here should be implemented consistently across the home app, signaling server, and any future admin or monitoring tooling.
- The existing `calls` and `audit_logs` DB tables should be the source of truth for call history; the new `call_delivery_attempts` table is an operational detail layer.