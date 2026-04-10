# Step 11-A2 — Reactive Stale Token Cleanup (11.14)

## Problem

When a push notification fails because a device token is invalid (device wiped, app reinstalled, token expired), the server only logs the error. The dead token stays in `device_tokens` and silently absorbs delivery attempts on every subsequent ring.

## Dependencies

None — standalone.

## What To Do

### Prune tokens on push failure (`wsHandler.js`)

In the ring push-sending loop (lines ~148–179):

- **FCM:** in the `.catch()` block, inspect the Firebase error code. If the code is `messaging/registration-token-not-registered` or `messaging/invalid-registration-token`, delete the token immediately:
  ```sql
  DELETE FROM device_tokens WHERE token = $1 AND token_type = 'fcm'
  ```
- **APNs (VoIP):** `sendVoipPush()` already returns `{ success, reason }`. After the call, if `reason` is `BadDeviceToken` or `Unregistered`, delete the token:
  ```sql
  DELETE FROM device_tokens WHERE token = $1 AND token_type = 'voip'
  ```

Apply the same cleanup in the `call-taken` push block (lines ~235–264).

### Clean old tokens on registration (`httpRoutes.js`)

When `/register-fcm-token` or `/register-voip-token` receives a new token for a user, delete any previous tokens for that same `(user_id, token_type)` that have a different token value:

```sql
DELETE FROM device_tokens
WHERE user_id = $1 AND token_type = $2 AND token != $3
```

The current `ON CONFLICT` upsert handles same-token updates, but if the token itself changes (new install), old tokens linger.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/wsHandler.js` | Add token deletion in FCM `.catch()` and after VoIP push failure, in both ring and call-taken blocks |
| `vidrom-signaling-server/src/httpRoutes.js` | Delete old tokens for same `(user_id, token_type)` on registration |

## Verification

- [ ] FCM push to invalid token → token row deleted from `device_tokens`
- [ ] VoIP push returning `BadDeviceToken` → token row deleted from `device_tokens`
- [ ] VoIP push returning `Unregistered` → token row deleted from `device_tokens`
- [ ] Same cleanup applies in the `call-taken` push path, not just ring
- [ ] Registering a new FCM token deletes any old FCM tokens for the same user
- [ ] Registering a new VoIP token deletes any old VoIP tokens for the same user
- [ ] Valid tokens are never deleted (only specific error codes trigger cleanup)
