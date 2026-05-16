# Step 12-B1 — DB And TURN Hardening (12.4)

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