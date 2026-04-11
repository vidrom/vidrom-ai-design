# C4 — Fallback Escalation Channels (11.8)

## Context

Part of Step 11 Phase C — Production Hardening.

**Dependencies:** C3 (health data), Phase B (retry orchestration, delivery-degraded events)
**Effort:** Large

---

## Problem

If no device in the apartment acknowledges a ring after all retries (the `delivery-degraded` event from Phase B — 11.4), the visitor is stuck. The intercom shows "No response" and the call ends. There is no alternative way to reach the resident.

This is the last line of defense — it only fires when the entire push delivery pipeline has failed for a specific call.

## What To Do

### Database — escalation configuration

Create migration `010-escalation-config.sql`:

```sql
CREATE TABLE IF NOT EXISTS escalation_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    apartment_id    UUID NOT NULL REFERENCES apartments(id) ON DELETE CASCADE,
    channel         VARCHAR(20) NOT NULL CHECK (channel IN ('sms', 'phone-call', 'concierge')),
    priority        INTEGER NOT NULL DEFAULT 1,
    target          TEXT NOT NULL,  -- phone number, concierge user ID, etc.
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    UNIQUE (apartment_id, channel, target)
);

CREATE INDEX idx_escalation_configs_apartment ON escalation_configs (apartment_id);
```

Each apartment can have multiple escalation channels ordered by priority. For example:

| Priority | Channel | Target |
|----------|---------|--------|
| 1 | sms | +972501234567 |
| 2 | phone-call | +972501234567 |
| 3 | concierge | (building concierge user ID) |

### Escalation orchestrator

Add a new module `escalationOrchestrator.js`:

**File: `vidrom-signaling-server/src/escalationOrchestrator.js`**

Triggered when the retry orchestrator fires the `delivery-degraded` event (no device acked after all retries):

1. Query `escalation_configs` for the apartment, ordered by priority
2. For each enabled channel, attempt escalation:

**SMS escalation:**
- Send an SMS via a third-party API (e.g., Twilio, AWS SNS) with a message like: "Someone is at your door at [Building Name]. Open the Vidrom app to answer."
- Include a deep link: `vidrom://call/{callId}`
- Log the attempt in `audit_logs` with event type `escalation-sms-sent`

**Phone call escalation:**
- Place an automated call via a telephony API with a brief message: "You have a visitor at [Building Name]. Please open the Vidrom app."
- Log as `escalation-phone-sent`

**Concierge escalation:**
- Send a push notification or WS message to the concierge user/device
- The concierge can answer the intercom call on behalf of the apartment or contact the resident by other means
- Log as `escalation-concierge-sent`

3. Track escalation attempts in the `call_delivery_attempts` table with a new delivery state: `escalated`
4. If escalation succeeds (resident opens app and accepts), the normal accept flow handles it
5. If all escalation channels fail or time out, log `escalation-failed` in `audit_logs`

### Server — integrate with retry orchestrator

**File: `vidrom-signaling-server/src/retryOrchestrator.js`**

In the `checkDeliveryDegraded` function (the t=15s final check):

```javascript
// Existing: log delivery-degraded, send ring-progress noResponse
// New: trigger escalation if configured
const configs = await getEscalationConfigs(apartmentId);
if (configs.length > 0) {
  await triggerEscalation(callId, apartmentId, configs);
}
```

The escalation should only fire if the ring is still active (not yet expired or answered). Check `calls.status = 'calling'` before escalating.

### Management portal — escalation configuration

**File: `vidrom-signaling-server/portals/management.html`**

Add an "Escalation Settings" section per apartment where building managers can:

- Add/remove escalation channels (SMS, phone, concierge)
- Set phone numbers for SMS and phone escalation
- Enable/disable channels
- Set priority order

**File: `vidrom-signaling-server/lambda/managementRoutes.js`**

Add endpoints:
- **GET `/api/management/apartments/:apartmentId/escalation`** — get escalation configs
- **PUT `/api/management/apartments/:apartmentId/escalation`** — update escalation configs

### Intercom — show escalation status

When escalation is triggered, send a `ring-progress` update to the intercom:

```json
{ "type": "ring-progress", "callId": "...", "escalating": true, "channel": "sms" }
```

The intercom can show: "Attempting to reach resident via SMS..."

This gives the visitor hope that someone is still being contacted, rather than just "No response."

### Rate limiting and safety

- Do not escalate the same apartment more than once per call
- Do not escalate more than 3 times per apartment per hour (prevent abuse from repeated intercom rings)
- Do not send SMS or phone calls to the same number more than 5 times per day
- Log all escalation attempts for audit

## Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/sql/010-escalation-config.sql` | New migration — escalation configuration table |
| `vidrom-signaling-server/src/escalationOrchestrator.js` | New module — escalation channel execution |
| `vidrom-signaling-server/src/retryOrchestrator.js` | Trigger escalation on delivery-degraded |
| `vidrom-signaling-server/src/wsHandler.js` | Send `ring-progress` with escalation status to intercom |
| `vidrom-signaling-server/lambda/managementRoutes.js` | Escalation config CRUD endpoints |
| `vidrom-signaling-server/portals/management.html` | Escalation settings UI per apartment |
| `vidrom-ai-intercom/InCallScreen.js` | Display escalation status to visitor |

## Third-Party Dependencies

Escalation channels require integration with external services:

| Channel | Suggested Service | Notes |
|---------|-------------------|-------|
| SMS | AWS SNS or Twilio | AWS SNS is simpler if already on AWS; Twilio has better delivery reporting |
| Phone call | Twilio | Programmable voice with TTS message |
| Concierge | Internal (WS/push) | No external dependency |

Start with SMS only (highest value, simplest integration), then add phone and concierge as needed.

## Verification

- [ ] `escalation_configs` table stores per-apartment escalation channels
- [ ] Management portal allows configuring escalation channels
- [ ] When delivery-degraded fires and escalation is configured, SMS is sent
- [ ] SMS contains building name and deep link
- [ ] Escalation is logged in `audit_logs`
- [ ] Escalation does not fire if the call was already answered or expired
- [ ] Rate limits prevent excessive escalation (max 3/hour per apartment, 5/day per number)
- [ ] Intercom shows "Attempting to reach resident via SMS..." during escalation
- [ ] Apartments without escalation configs get the existing no-response behavior (no change)
