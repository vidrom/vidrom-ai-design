# Step 13-A3 — Authenticated Token Registration

## Problem

`/register-fcm-token` and `/register-voip-token` currently trust caller-supplied `userId` and `apartmentId`. That enables token hijacking, cross-apartment poisoning, and deletion of another resident's token rows.

## Dependencies

Requires [step13-A1-resident-auth-foundation.md](step13-A1-resident-auth-foundation.md).

## What To Do

### Server

For both resident token registration endpoints:

1. Require resident bearer auth.
2. Accept only device facts from the client, such as `token` and `platform`.
3. Derive `userId` and apartment scope from authenticated resident context.
4. Upsert `device_tokens` only for that resident.
5. Delete only stale tokens owned by that same resident and token type.
6. Keep `device_health` updates, but only after authorization passes.

### Contract

Prefer bodies shaped like:

```json
{ "token": "...", "platform": "ios|android" }
```

for FCM, and:

```json
{ "token": "..." }
```

for VoIP.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/httpRoutes.js` | Enforce resident auth and server-derived token ownership |
| `vidrom-ai-home/fcmService.js` | Stop sending ownership fields as authority input |
| `vidrom-ai-home/voipService.js` | Stop sending ownership fields as authority input |

## Verification

- [ ] token registration returns `401` without bearer auth
- [ ] resident cannot bind a token to another apartment
- [ ] resident cannot delete another resident's active token rows
- [ ] authorized resident token registration still populates `device_tokens`
- [ ] authorized resident token registration still updates `device_health`