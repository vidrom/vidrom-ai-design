# Step 16 Final — Production Cutover

## Scope

Run this final Step 16 task after the local Step 16 implementation work is ready for production rollout.

Local code and design changes can be completed independently, but production verification, secret rotation, and live infrastructure deployment happen here.

## Prerequisites

- Step 16 local implementation is complete
- server, mobile, intercom, Lambda, and CDK tests pass locally
- dependency updates are committed or explicitly documented as accepted risk
- manual testing has not started yet, or is restarted after this cutover

Current verification baseline from the latest Step 16 review:

- local validation already passed for signaling server, CDK, Home app, and Intercom app
- production already returns `200` on `/healthz`
- production already returns `401` on `/debug/status` without admin auth
- production already returns `401` on `/api/rtc-config` without client auth
- production ALB health checks already point to `/healthz`
- production WAF is currently removed from the signaling ALB and portal CloudFront distribution as an approved cost-control step; see the note below and [pre-production-waf-reintroduction.md](../todo-future/pre-production-waf-reintroduction.md)
- production RDS already has encryption, deletion protection, backup retention, and private access enabled

## Current Production Note

As of 2026-06-18, the production signaling ALB WAF and portal CloudFront WAF were intentionally removed to reduce recurring AWS cost while traffic remains low.

This is an operational cost decision, not the final intended hardened production posture.

The approved live change included:

1. removal of the regional signaling ALB WAF
2. removal of the signaling ALB WAF association
3. removal of the portal CloudFront WAF
4. removal of the portal distribution `WebACLId`

Live verification after the change confirmed:

- `https://signaling.vidrom.com/healthz` returned `200` after recovery
- `https://portal.vidrom.com/api/admin/buildings` still returned `401` without auth
- `https://vidrom.com/` still returned `200`
- `https://vidrom.co.il/` still returned `200`

Operational note:

- the stack update also replaced the signaling EC2 instance because the deployed diff included a `UserData` change unrelated to WAF
- this temporarily caused `502` responses from `signaling.vidrom.com`
- service was recovered by re-running the canonical EC2 deploy path with `vidrom-cdk/deploy-server-ssm.sh`

Before calling the environment production-hardened again, complete [pre-production-waf-reintroduction.md](../todo-future/pre-production-waf-reintroduction.md).

## Tasks

1. **Deploy CDK hardening changes**
   - deploy any remaining stack deltas
   - verify ALB health checks use `/healthz`
   - verify `/debug/status` is no longer publicly readable
   - verify SSH is not publicly exposed and SSM Session Manager still works
   - verify RDS deletion protection/encryption/retention settings
   - verify portal bucket retention behavior is appropriate for production
   - if the stack diff would replace EC2, plan for an immediate follow-up `./deploy-server-ssm.sh` so the new instance receives the signaling app bundle

2. **Deploy signaling server changes**
   - update EC2 application files and dependencies only if the deployed instance is behind the verified local workspace state
   - restart `vidrom-signaling.service`
   - verify the host no longer runs or exposes the legacy `coturn` service
   - verify `NODE_ENV=production` startup still passes validation

3. **Deploy portal Lambda changes**
   - deploy Lambda API bundle with door-code response changes and dependency updates
   - verify admin and management portal auth still works
   - verify portal device lists do not expose plaintext door codes or imply that existing door codes are readable

4. **Run production secret rotation / restriction**
   - rotate JWT secret if production devices can be reprovisioned safely
   - rotate Twilio runtime credentials if the current values were used during development
   - rotate APNs auth key if any key material was exposed during development
   - rotate Firebase service-account secret if needed
   - restrict Firebase mobile API keys by bundle ID, Android SHA fingerprints, and allowed APIs
   - if WAF was removed for cost control, complete [pre-production-waf-reintroduction.md](../todo-future/pre-production-waf-reintroduction.md) before treating the environment as production-hardened

5. **Run live smoke tests**
   - Home login and apartment resolution
   - intercom provisioning
   - intercom reconnect with stored JWT
   - apartment list and building info on intercom
   - incoming call, accept, decline, hangup, and open-door
   - watch mode
   - iOS VoIP push and Android FCM wakeup paths
   - RTC config retrieval from authenticated Home and Intercom clients
   - debug/status access denied without admin auth
   - oversized / noisy requests rejected or rate-limited

## Remaining Focus

This cutover task is now mostly about operational proof rather than implementation rollout. Prioritize:

1. secret rotation and restriction
2. authenticated end-to-end call-flow and push smoke tests
3. confirming the deployed portal bundle matches the write-only door-code UI behavior
4. documenting any dependency advisories that are being intentionally deferred

6. **Prepare for manual testing**
   - confirm no live CloudWatch/server logs contain bearer tokens, door codes, or raw push tokens
   - confirm `npm audit --omit=dev` status for deployed server/Lambda packages
   - confirm any remaining dependency findings are documented with risk and revisit date
   - restart the manual testing plan from the beginning if any call-flow/security behavior changed during deployment

## Acceptance Criteria

- production deployment is successful
- `/healthz` is public and minimal
- `/debug/status` requires authorized admin access or is unavailable in production
- Home WebSocket auth and intercom JWT auth both work against production
- public provisioning and RTC config endpoints are rate-limited and/or authenticated
- door-code verification never returns expected codes
- production secrets are rotated or explicitly deferred with a written reason
- manual testing can begin against the hardened production-like environment
