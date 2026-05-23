# Step 13-A4 — Authenticated Call Actions

## Problem

The resident HTTP paths for delivery ack and HTTP accept currently trust caller-supplied user identity. That allows forged call telemetry and unauthorized call claims.

## Dependencies

Requires [step13-A1-resident-auth-foundation.md](step13-A1-resident-auth-foundation.md).

## What To Do

### Delivery ack

For `POST /api/home/calls/:callId/ack`:

1. Require resident bearer auth.
2. Load the call by `callId`.
3. Verify the call belongs to one of the authenticated resident's apartments.
4. Accept only allowed ack events.
5. Bind the ack to a token owned by that resident instead of trusting `userId` from the body.

### HTTP accept

For `POST /api/home/calls/:callId/accept`:

1. Require resident bearer auth.
2. Load the call by `callId`.
3. Verify the call apartment belongs to the authenticated resident.
4. Reuse the existing atomic accept logic only after authorization passes.
5. Use the server-trusted resident `userId` in audit logs and in-memory reservation state.

The fast accept path from Step 11-A6 should remain intact; only the authority model changes.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/httpRoutes.js` | Enforce resident auth and apartment ownership for ack and accept |
| `vidrom-ai-home/useCallManager.js` | Stop sending `userId` as authority input on accept and ack |

## Verification

- [ ] call ack returns `401` without bearer auth
- [ ] call ack rejects calls outside the resident's apartment scope
- [ ] HTTP accept returns `401` without bearer auth
- [ ] HTTP accept rejects calls outside the resident's apartment scope
- [ ] legitimate resident can still accept a real ringing call
- [ ] delivery and audit data still record correctly for authorized flows