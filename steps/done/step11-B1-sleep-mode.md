# Step 11-B1 — Respect Per-User Sleep Mode in Delivery Workflow (11.12)

## Problem

The `users` table already has a `sleep_mode` boolean, but the ring handler in `wsHandler.js` does not check it. Push notifications are sent to all `device_tokens` for the apartment regardless. A sleeping resident gets woken up, and a visitor rings into the void if all residents have sleep mode on.

## Dependencies

Phase A (callId in place). No other Phase B dependencies — standalone within Phase B.

## What To Do

### Server — ring handler (`wsHandler.js`)

When the intercom sends a `ring`, **before** sending push notifications:

1. Query users with sleep mode status for the target apartment:
   ```sql
   SELECT u.id, u.sleep_mode
   FROM users u
   JOIN apartment_residents ar ON ar.user_id = u.id
   WHERE ar.apartment_id = $1
   ```
2. Collect the set of sleeping user IDs.
3. If **all** residents have `sleep_mode = true`, do not ring at all:
   - Do not create a `calls` row with status `calling`
   - Send the intercom an immediate response: `{ type: 'apartment-unavailable', reason: 'all-residents-sleeping', apartmentId }`
   - Break out of the ring handler
4. If **some** residents are sleeping, filter the push token query to exclude sleeping users:
   ```sql
   SELECT dt.token, dt.token_type, dt.platform
   FROM device_tokens dt
   WHERE dt.apartment_id = $1
     AND dt.user_id NOT IN (SELECT u.id FROM users u WHERE u.sleep_mode = true AND u.id = dt.user_id)
   ```
   Or simpler: join with users and filter `WHERE u.sleep_mode = false`.
5. WebSocket ring messages to connected home clients should also be skipped for sleeping users. This requires knowing which connected home client belongs to which user. If user identity is not tracked on the WS connection, send the ring to all connected WS clients but let the home app handle sleep mode locally (lower priority — push filtering is the main win).

### Intercom — `InCallScreen.js`

Handle the new `apartment-unavailable` message:
- If `reason === 'all-residents-sleeping'`, display "Residents are currently unavailable" instead of "Ringing..."
- After a short delay (3–5 seconds), return to the apartment list or home screen

### Audit

- Log skipped deliveries in `audit_logs` with event type `ring-skipped-sleep-mode` so the team knows how often this fires.

## Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/wsHandler.js` | Query sleep mode before push fanout; send `apartment-unavailable` if all sleeping |
| `vidrom-ai-intercom/InCallScreen.js` | Handle `apartment-unavailable` WS message |

## Verification

- [ ] If all residents have `sleep_mode = true`, intercom receives `apartment-unavailable` immediately
- [ ] If some residents are sleeping, only awake residents' devices receive push
- [ ] `audit_logs` records `ring-skipped-sleep-mode` when applicable
- [ ] Normal ringing is unaffected when no residents are in sleep mode
