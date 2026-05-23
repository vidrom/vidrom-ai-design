# Step 13-B1 — Home Bearer Auth Transport

## Problem

Even after the server is hardened, the home app still needs a consistent way to attach authenticated resident bearer tokens to every resident-facing signaling HTTP request.

## Dependencies

Best landed alongside:

- [step13-A2-authenticated-apartment-resolution.md](step13-A2-authenticated-apartment-resolution.md)
- [step13-A3-authenticated-token-registration.md](step13-A3-authenticated-token-registration.md)
- [step13-A4-authenticated-call-actions.md](step13-A4-authenticated-call-actions.md)

## What To Do

### Home app

1. Add a helper in the auth layer to fetch the current Firebase ID token.
2. Centralize bearer-header creation for resident HTTP requests.
3. Attach `Authorization: Bearer <firebase-id-token>` to:
   - apartment resolution
   - FCM token registration
   - VoIP token registration
   - delivery ack
   - HTTP accept
4. Let Firebase handle token refresh by requesting the current token when needed.

### Cleanup

Reduce duplicate request-header construction across the home app.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-ai-home/authService.js` | Expose Firebase ID token helper |
| `vidrom-ai-home/HomeScreen.js` | Send bearer auth on apartment resolution |
| `vidrom-ai-home/fcmService.js` | Send bearer auth on FCM registration |
| `vidrom-ai-home/voipService.js` | Send bearer auth on VoIP registration |
| `vidrom-ai-home/useCallManager.js` and/or `config.js` | Send bearer auth on ack and accept |

## Verification

- [ ] all resident HTTP requests send bearer auth
- [ ] apartment resolution still succeeds after auth headers are required
- [ ] FCM and VoIP token registration still succeed for a legitimate resident
- [ ] HTTP accept and ack still work with bearer auth attached