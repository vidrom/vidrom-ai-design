# Step 11-A3 — Building-Level No-Answer Timeout (11.13)

## Problem

The server uses a hardcoded `30000` ms timeout in `connectionState.js` for pending rings. The database already has a `no_answer_timeout` column on the `buildings` table, but it is not used. Building managers cannot tune ring behavior per building without code changes.

## Dependencies

None — standalone.

## What To Do

### Accept timeout parameter (`connectionState.js`)

The `setPendingRing()` function currently hardcodes `30000` ms. Change it to accept a `timeoutMs` parameter:

```javascript
function setPendingRing(apartmentId, intercomDeviceId, timeoutMs = 30000) {
```

### Query building timeout on ring (`wsHandler.js`)

Before calling `setPendingRing()` during a `ring`, query the building's `no_answer_timeout`:

```sql
SELECT b.no_answer_timeout
FROM buildings b
JOIN apartments a ON a.building_id = b.id
WHERE a.id = $1
```

If the value exists, convert it to milliseconds and pass it. If not, fall back to the `global_settings` table's value, then to the default 30000 ms.

### Match VoIP push TTL (`apnsService.js`)

The VoIP push TTL is currently hardcoded to 30 s (line ~40). Accept an optional TTL parameter in `sendVoipPush()` so the push expiry matches the ring lifetime:

```javascript
async function sendVoipPush(voipToken, callerName, payloadOverride, ttlSeconds = 30) {
```

Pass the building timeout (in seconds) when calling `sendVoipPush()` from the ring handler.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/connectionState.js` | Accept `timeoutMs` param in `setPendingRing()` |
| `vidrom-signaling-server/src/wsHandler.js` | Query building timeout before `setPendingRing()` and pass to `sendVoipPush()` |
| `vidrom-signaling-server/src/apnsService.js` | Accept optional TTL param in `sendVoipPush()` |

## Verification

- [ ] Building with custom `no_answer_timeout` (e.g. 45 s) → ring lasts 45 s, not 30 s
- [ ] Building with no `no_answer_timeout` set → falls back to `global_settings`, then 30 s default
- [ ] VoIP push TTL matches the building's timeout value
- [ ] FCM push TTL matches the building's timeout value (if FCM supports TTL configuration)
- [ ] Ring expiry log message reflects the actual timeout used, not always "30s"
