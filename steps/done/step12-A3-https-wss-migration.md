# Step 12-A3 — HTTPS And WSS Migration (12.3)

## Problem

The apps still target plain HTTP and plain WebSocket endpoints, and iOS is configured to allow insecure HTTP loads for the signaling host.

That is not acceptable for a production calling stack.

## Dependencies

Recommended after A2 so the secure endpoint configuration lands on top of the validated config path.

## What To Do

### Infrastructure

1. Put the signaling endpoint behind TLS.
2. Expose secure client endpoints via `https://` and `wss://`.
3. Decide whether TLS terminates at a load balancer or reverse proxy, or directly in the Node server.

### Mobile apps

1. Update runtime config to use `https://` and `wss://`.
2. Remove the ATS insecure-load exception from the home app once secure transport is live.
3. Keep local development overrides separate from production defaults.

### Docs and ops

1. Update outputs, runbooks, and start scripts to reference secure endpoints.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-ai-home/config.js` | Switch production endpoints to HTTPS/WSS |
| `vidrom-ai-home/app.json` | Remove production ATS insecure transport allowance when no longer needed |
| `vidrom-ai-intercom/config.js` | Switch production endpoints to HTTPS/WSS |
| `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Provision or describe the TLS entry point |
| `vidrom-cdk/start-instances.sh` | Stop advertising or printing insecure endpoints |
| design docs that mention signaling endpoints | Update examples and runbooks to secure URLs |

## Verification

- [ ] Mobile apps connect successfully over HTTPS and WSS
- [ ] iOS no longer needs insecure transport exceptions for the production host
- [x] Docs and scripts show secure endpoints only

## Current Status

- Local implementation is complete.
- The remaining production work is tracked in [step12-A3-final-production-cutover.md](step12-A3-final-production-cutover.md).

## Implementation Notes

- Production client defaults now target `https://signaling.vidrom.com` and `wss://signaling.vidrom.com`.
- Local and temporary non-production endpoint testing should use `EXPO_PUBLIC_SIGNALING_HTTP_URL` and `EXPO_PUBLIC_SIGNALING_WS_URL` when starting Metro instead of editing app source.
- CDK implementation uses an ACM certificate plus internet-facing ALB for TLS termination in front of the existing Node backend on port `8080`.
- AWS deploy is pending because the account is currently blocked; infra changes have been implemented and validated with `npm run build` and `cdk synth` in `vidrom-cdk`.