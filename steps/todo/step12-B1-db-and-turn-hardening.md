# Step 12-B1 — DB And TURN Hardening (12.4)

## Status

Implemented in code, pending deploy verification.

The repo changes for DB hardening and runtime TURN credentials have landed, but this step should stay in `todo/` until the deployed AWS environment is verified after the account/deploy blocker is cleared.

## Problem

The current infra model exposes more surface area than necessary:

- RDS is publicly accessible
- database ingress is broader than ideal
- TURN uses static credentials

## Dependencies

A2 should land first so secret storage and runtime TURN config are already in place.

## What To Do

### Database

1. Move RDS access behind a tighter boundary.
2. Remove broad public ingress for PostgreSQL.
3. Prefer application-layer access from trusted components only.
4. Revisit whether Lambda should reach DB directly or through a controlled private path.

### TURN

1. Replace static TURN credentials with time-limited credentials.
2. Generate credentials per session or short interval.
3. Expose TURN config from the server to clients instead of baking it into source.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Tighten DB network exposure and security groups |
| signaling server config and ICE config endpoint | Generate or broker short-lived TURN credentials |
| mobile config consumers | Use runtime TURN config instead of static source constants |

## Verification

- [ ] DB is no longer broadly reachable from the public internet
- [ ] TURN credentials rotate or expire automatically
- [ ] Clients can still establish calls in restrictive network conditions

## Post-Deploy Verification Checklist

1. Deploy the CDK stack and confirm the update completes successfully.
2. Confirm the RDS instance is not publicly accessible and only trusted security groups can reach PostgreSQL.
3. Confirm the signaling service still serves `/api/rtc-config` successfully in production.
4. Inspect a production `/api/rtc-config` response and verify TURN credentials include an expiry and are not static.
5. Place a real intercom-to-home call and verify call setup still succeeds.
6. Re-test from a restrictive network path where TURN relay may be required and confirm media still connects.
7. If all checks pass, move this step to `done/` and mark the verification items complete.