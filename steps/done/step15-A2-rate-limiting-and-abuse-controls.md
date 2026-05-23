# Step 15-A2 — Rate Limiting And Abuse Controls

## Problem

The portal API now has tighter CORS and security headers, but it still accepts repeated high-rate request bursts without any explicit application-layer abuse controls.

That leaves two practical gaps:

1. repeated unauthorized or noisy requests can still burn Lambda and token-verification work
2. sensitive write routes like device provisioning, revoke, reprovision, and other admin mutations can be hammered more aggressively than they should be

## What To Do

Add two layers of throttling for the portal API:

1. API Gateway default-route throttling to absorb broad bursts before they reach Lambda
2. Lambda-side fixed-window rate limits that return deterministic `429` responses with `Retry-After`

Within Lambda:

1. apply a per-IP request budget to `/api/admin/*` and `/api/management/*` before auth succeeds
2. apply stricter per-operator write limits after auth succeeds
3. apply the tightest limits to provisioning-sensitive routes:
   - `POST /api/admin/devices`
   - `POST /api/admin/devices/:id/revoke`
   - `POST /api/admin/devices/:id/reprovision`
   - management equivalents
4. keep thresholds configurable through environment variables so production tuning does not require code changes

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/lambda/rateLimit.js` | Add fixed-window rate-limit helpers |
| `vidrom-signaling-server/lambda/handler.js` | Enforce portal request and mutation limits |
| `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Add API Gateway stage throttling |
| portal Lambda tests | Add `429` regression coverage |

## Verification

- [ ] repeated unauthorized portal requests eventually return `429`
- [ ] repeated provisioning requests from the same operator eventually return `429`
- [ ] throttled responses include `Retry-After`
- [ ] normal portal reads still succeed under ordinary usage
- [ ] CDK synth/build still succeeds with the new stage throttling