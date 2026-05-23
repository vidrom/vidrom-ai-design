# Step 15-A1 — Portal CORS And Security Headers

## Problem

The portal Lambda/API layer currently returns permissive CORS headers and a minimal response-header set.

That leaves two unnecessary weaknesses:

1. browser requests are not restricted to the intended portal origin
2. API responses do not consistently carry defensive browser-facing headers like HSTS, nosniff, frame restrictions, and cache suppression

## What To Do

For portal API responses under `/api/admin/*` and `/api/management/*`:

1. allow only the intended portal origin by default
2. support explicit origin overrides through configuration when needed for local testing
3. add `Vary: Origin`
4. add `Strict-Transport-Security`
5. add `X-Content-Type-Options: nosniff`
6. add `X-Frame-Options: DENY`
7. add a restrictive API-appropriate `Content-Security-Policy`
8. return `Cache-Control: no-store` for authenticated portal API responses

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/lambda/handler.js` | Restrict CORS and add security headers |
| `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Restrict API Gateway CORS preflight to the portal origin |
| portal Lambda tests | Add response-header regression coverage |

## Verification

- [ ] browser requests from `https://portal.vidrom.com` still succeed
- [ ] unknown origins do not receive permissive CORS headers
- [ ] API responses include the expected security headers
- [ ] OPTIONS preflight remains functional for the intended origin