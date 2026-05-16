# Step 12-A2 — Secret Management And Startup Validation (12.2)

## Problem

Several secrets or sensitive identifiers are embedded directly in committed code or shell scripts:

- JWT fallback secret
- DB password defaults
- TURN credentials
- APNs identifiers in helper scripts and deployment flow

This weakens security and also makes configuration drift harder to detect.

## Dependencies

None — standalone, but it should land before transport and infra hardening that reuse the same config paths.

## What To Do

### Signaling server

In `vidrom-signaling-server/src/auth.js`:

1. Remove the fallback JWT secret.
2. Require `JWT_SECRET` at startup.
3. Fail process startup with a clear error if it is missing.

In `vidrom-signaling-server/src/server.js` and related startup/config modules:

1. Centralize required environment validation.
2. Treat Firebase admin credentials, APNs config, and DB credentials as startup requirements for production.

### Mobile apps

In `vidrom-ai-home` and `vidrom-ai-intercom`:

1. Remove embedded TURN username and password from shipped source.
2. Fetch ICE/TURN configuration from the signaling backend at runtime, ideally with short-lived credentials.

### CDK and deploy scripts

In `vidrom-cdk`:

1. Move DB credentials and other sensitive configuration to AWS Secrets Manager or SSM Parameter Store.
2. Stop templating secrets directly into systemd unit files.
3. Keep deploy scripts focused on artifact delivery, not secret definition.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/auth.js` | Remove JWT fallback and require env-backed secret |
| `vidrom-signaling-server/src/server.js` | Centralize required config validation |
| `vidrom-signaling-server/src/apnsService.js` | Ensure APNs config comes from managed env/config |
| `vidrom-ai-home/config.js` | Remove embedded TURN credentials |
| `vidrom-ai-intercom/webrtcHtml.js` | Remove embedded TURN credentials |
| `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Source secrets from AWS managed storage |
| `vidrom-cdk/deploy-server-ssm.sh` | Stop injecting raw secret values into deploy flow |
| `vidrom-signaling-server/run.sh` | Read required config without defining secrets inline |

## Verification

- [ ] Server refuses to start if required secrets are missing
- [ ] No long-lived app credentials remain embedded in mobile source
- [ ] Production secrets come from managed secret storage