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
- production WAF is already attached to the signaling ALB
- production RDS already has encryption, deletion protection, backup retention, and private access enabled

## Tasks

1. **Deploy CDK hardening changes**
   - deploy any remaining stack deltas
   - verify ALB health checks use `/healthz`
   - verify `/debug/status` is no longer publicly readable
   - verify SSH is not publicly exposed and SSM Session Manager still works
   - verify RDS deletion protection/encryption/retention settings
   - verify portal bucket retention behavior is appropriate for production

2. **Deploy signaling server changes**
   - update EC2 application files and dependencies only if the deployed instance is behind the verified local workspace state
   - restart `vidrom-signaling.service`
   - restart coturn if runtime TURN settings changed
   - verify `NODE_ENV=production` startup still passes validation

3. **Deploy portal Lambda changes**
   - deploy Lambda API bundle with door-code response changes and dependency updates
   - verify admin and management portal auth still works
   - verify portal device lists do not expose plaintext door codes or imply that existing door codes are readable

4. **Run production secret rotation / restriction**
   - rotate JWT secret if production devices can be reprovisioned safely
   - rotate TURN shared secret and restart both signaling and coturn
   - rotate APNs auth key if any key material was exposed during development
   - rotate Firebase service-account secret if needed
   - restrict Firebase mobile API keys by bundle ID, Android SHA fingerprints, and allowed APIs

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
