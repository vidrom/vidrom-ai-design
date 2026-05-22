# Step 13-A2 — Authenticated Apartment Resolution

## Problem

`POST /api/home/resolve-apartment` currently trusts client-supplied email to determine resident identity. That allows spoofing and apartment enumeration.

## Dependencies

Requires [step13-A1-resident-auth-foundation.md](step13-A1-resident-auth-foundation.md).

## What To Do

### Server

For `POST /api/home/resolve-apartment`:

1. Require resident bearer auth.
2. Ignore client-supplied email for identity.
3. Resolve apartment membership from the authenticated resident context.
4. Return only apartments owned by that resident.
5. If the app still assumes a single apartment, keep returning the primary apartment in the current shape.

### Compatibility

The route can keep the same name, but the request body should become optional and ignored for authorization.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/httpRoutes.js` | Change apartment resolution to use authenticated resident context only |
| `vidrom-ai-home/HomeScreen.js` | Stop relying on body email as authority input |

## Verification

- [ ] endpoint returns `401` without bearer auth
- [ ] spoofed email in the body does not change the result
- [ ] authenticated resident receives only their own apartment data
- [ ] existing home-app bootstrap still works for a legitimate resident