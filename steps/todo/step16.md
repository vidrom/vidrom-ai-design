# Step 16 — Security Review Remediation

## Scope

Step 16 captures the security improvements identified during the May 23, 2026 repository security review.

The highest remaining gaps are in the signaling server and runtime edge rather than the portal identity model. Admin and management portal auth, portal CORS/security headers, and portal rate limiting are already mostly in place from Step 15. The next priority is to harden Home WebSocket identity, public EC2 endpoints, call/control message authorization, request abuse limits, dependency posture, and infrastructure blast radius.

## Goals

After Step 16:

1. Home WebSocket sessions are authenticated with Firebase ID tokens and authorized against server-resolved apartments
2. call-control WebSocket messages such as `accept`, `decline`, `watch`, `open-door`, `hangup`, SDP, and ICE candidates cannot be sent by unauthenticated or apartment-spoofed clients
3. door-code verification never discloses expected codes, and door codes are no longer stored or compared as plaintext
4. public debug/status output is separated from load-balancer health checks and protected behind admin-only access
5. high-risk public endpoints have body-size limits, rate limits, and stricter input validation
6. unauthenticated TURN credential issuance is removed or tightly constrained
7. production dependency vulnerabilities are upgraded or explicitly accepted with documented risk
8. infrastructure defaults reduce internet exposure, IAM scope, and destructive production settings
9. repo and production secrets are rotated or constrained before production cutover

## Current Status

Local implementation is in progress and the main Step 16 hardening set has been applied in the workspace. Live AWS deployment, production secret rotation, and production smoke tests remain deferred until the AWS account is unblocked.

Decision: implement the full Step 16 hardening set before starting the manual testing stage, rather than deferring lower-priority security items until after manual QA.

Rationale:

- manual testing should validate the final security posture and call-flow behavior, not a partially hardened interim state
- Home WebSocket auth and call-control authorization affect core flows, so they should be tested together with the rest of the call stack
- public endpoint hardening, dependency updates, and CDK/runtime changes can affect deployment and connectivity assumptions that manual testing should catch
- doing this as one security milestone reduces the risk of passing manual QA and then invalidating results with later auth/infra changes

Step 16 should still be implemented in small commits/slices internally, but all slices should land before the manual testing stage begins.

Step 16 should be completed before manual testing begins. The live AWS cutover tasks are tracked separately because the AWS account is temporarily blocked.

### Local Implementation Progress

Completed locally:

- Home WebSocket registration now requires a Firebase resident ID token and server-resolved apartment scope
- WebSocket call-control messages are gated by verified role, apartment, active call/watch state, and accepted participant state
- intercom device JWT registration requires the `intercom` role and matching active device/building context
- door-code verification no longer discloses expected codes and supports salted PBKDF2 hashes with plaintext runtime migration
- portal device listing/create/update paths no longer expose or persist new plaintext door codes
- `/healthz` is public/minimal and `/debug/status` is admin-protected with token-prefix disclosure removed
- `/api/rtc-config` requires resident or intercom device auth before returning ICE/TURN config
- EC2 routes now have JSON body-size caps, route-specific rate limits, client-error redaction/truncation, and stricter app validation
- intercom provisioning codes now require 6 digits, one-time pending state, and a configurable TTL
- CDK removes public SSH ingress, narrows EC2 S3 permissions, pins coturn, retains the portal bucket, enables RDS encryption/deletion protection, points ALB health checks to `/healthz`, and adds WAF managed/rate rules to the signaling ALB and portal CloudFront edge
- server, Lambda, CDK, Home, and Intercom lockfiles received non-breaking dependency audit updates where available
- regression tests now cover public health, protected debug status, authenticated RTC config, door-code non-disclosure, oversized-body rejection, and hardened resident call ownership

Still pending before production cutover:

- run `sql/012-door-code-hash.sql` in production after deployment planning
- deploy CDK/server/Lambda changes after AWS account access is restored
- rotate/restrict production JWT, TURN, APNs, Firebase, and mobile API-key material
- run live WSS/TURN/push/call-flow smoke tests before manual testing begins
- decide whether to accept remaining moderate dependency advisories or schedule breaking framework/package upgrades

Remaining `npm audit --omit=dev` findings after local safe fixes:

- signaling server: 8 moderate `uuid` transitive advisories through Firebase/Google optional dependency chains; force fix would install a breaking top-level `uuid`
- Lambda: 0 findings
- CDK: 1 moderate `brace-expansion` advisory bundled inside the latest `aws-cdk-lib`; `npm audit fix` cannot currently replace it
- Home app: 14 moderate advisories remaining through Expo/config chains; force fix would move to Expo 56
- Intercom app: 11 moderate advisories remaining through Expo/config chains; force fix would move to Expo 56

## Implementation Order

| Order | Focus | Primary Files | Notes |
|---|---|---|---|
| 1 | Authenticate Home WebSocket registration | `vidrom-ai-home/useCallManager.js`, `vidrom-signaling-server/src/wsHandler.js`, `vidrom-signaling-server/src/auth.js` | Include Firebase ID token in `register`; verify server-side; resolve authorized apartments from DB; ignore client-supplied apartment/user identity unless it matches the verified resident context |
| 2 | Authorize WebSocket call-control messages | `vidrom-signaling-server/src/wsHandler.js` | Gate `watch`, `accept`, `decline`, `open-door`, `hangup`, `offer`, `answer`, and `candidate` by verified role, apartment, active-call state, and accepted participant |
| 3 | Remove door-code disclosure and plaintext comparison | `vidrom-signaling-server/src/httpRoutes.js`, `vidrom-signaling-server/lambda/adminRoutes.js`, database migration | Return only `{ valid: false }` on failed verification; hash door codes with a slow password hash or HMAC; avoid showing stored door codes in portals unless explicitly required |
| 4 | Split public health from protected debug status | `vidrom-signaling-server/src/httpRoutes.js`, `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Add minimal unauthenticated `/healthz` for ALB health checks; protect `/debug/status` with admin auth or remove it from production |
| 5 | Add body-size limits and abuse controls to EC2 routes | `vidrom-signaling-server/src/httpRoutes.js` | Cap JSON request body size; validate content types; add per-IP/principal rate limits for provisioning, client errors, RTC config, and call actions |
| 6 | Harden `/api/client-error` ingestion | `vidrom-signaling-server/src/httpRoutes.js`, mobile `errorReporter.js` files | Require app/user/device auth where possible; truncate stack/context; redact tokens/emails; limit noisy clients |
| 7 | Protect or constrain `/api/rtc-config` | `vidrom-signaling-server/src/httpRoutes.js`, `vidrom-signaling-server/src/startupConfig.js`, mobile/intercom config callers | Prefer resident/device auth before issuing TURN credentials; otherwise add tight rate limits and short TTL/quotas |
| 8 | Harden intercom provisioning exchange | `vidrom-signaling-server/src/httpRoutes.js`, `vidrom-signaling-server/src/devices.js` | Add attempt counters, TTL enforcement checks, IP/code rate limiting, audit logs for failures, and one-time-use guarantees |
| 9 | Upgrade vulnerable production dependencies | all `package.json` / lockfiles | Prioritize server and Lambda; then mobile; then CDK. Address critical `protobufjs`, high XML/parser issues, `ws`, `node-forge` via `@parse/node-apn`, Expo/RN chain advisories, and Lambda `uuid` transitives |
| 10 | Infrastructure hardening | `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Remove world-open SSH or restrict to admin CIDR/SSM; narrow EC2 S3 permissions; enable explicit RDS encryption/deletion protection/backup policy; make portal bucket production-retained; add WAF/rate limits at ALB/CloudFront/API Gateway |
| 11 | Pin coturn build source | `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Avoid cloning an unpinned default branch; pin a release tag/commit and verify checksums or use a trusted package/image |
| 12 | Secret and repo hygiene follow-up | Firebase console, Apple Developer, Secrets Manager, repo files | Restrict Firebase mobile API keys by bundle ID/SHA/API scope; rotate production JWT/TURN/APNs/Firebase service-account material before production; remove or relocate APNs key metadata file if not needed |
| 13 | [step16-final-aws-unblocked-cutover.md](step16-final-aws-unblocked-cutover.md) | AWS-unblocked production deployment and verification | Run later, only after AWS account access is restored |

## AWS Blocked Follow-Up

The AWS account is currently blocked and expected to be restored in a few days. All local implementation work should proceed, but live deployment, production secret rotation, CloudFront/ALB/API Gateway verification, and live smoke tests are deferred to [step16-final-aws-unblocked-cutover.md](step16-final-aws-unblocked-cutover.md).

## Review Findings

### 1. Home WebSocket registration is not authenticated

Home clients currently register by sending `role: 'home'` plus an `apartmentId`. The server trusts that apartment ID for call routing and watch/call-control behavior.

Risk:

- a client can claim another apartment
- a client can receive ring events for that apartment
- a client can request watch mode
- a client can send call-control messages, SDP, ICE candidates, or `open-door`

Expected remediation:

- Home app obtains a Firebase ID token before opening/registering a WebSocket
- server verifies the token using Firebase Admin
- server resolves resident apartments from `users` / `apartment_residents` / `apartments`
- server accepts a requested apartment only if it belongs to the verified resident
- subsequent WS messages use server-side resident context, not client-supplied `userId`

### 2. Door-code verification leaks the expected code

Invalid door-code attempts currently return the expected code in the response. This should be removed immediately.

Expected remediation:

- failed verification returns only `{ valid: false }`
- successful verification returns only `{ valid: true }`
- door code storage is migrated away from plaintext
- portals should avoid displaying door codes by default

### 3. `/debug/status` is public and too detailed

The current debug route exposes active call state, intercom IDs, building IDs, connection state, APNs readiness, and token prefixes. It is also used for ALB health checks.

Expected remediation:

- add a minimal unauthenticated `/healthz` endpoint for ALB checks
- move debug detail behind admin auth, or disable it in production
- avoid returning token prefixes or raw internal IDs unless required for support and authorized

### 4. EC2 route request handling needs abuse limits

The EC2 HTTP handler reads request bodies without a size cap. Some routes are public or high-impact.

Expected remediation:

- reject oversized bodies before buffering too much memory
- require `application/json` on JSON routes
- add rate limits for provisioning, RTC config, client errors, and call-control fallback routes
- return consistent `413`, `415`, and `429` errors

### 5. `/api/client-error` can be abused as a write endpoint

The client-error endpoint accepts arbitrary unauthenticated payloads and writes them to the database.

Expected remediation:

- authenticate where practical
- cap and sanitize `message`, `stack`, and `context`
- redact tokens, emails, and IDs from client-provided fields
- add rate limiting and drop noisy clients

### 6. `/api/rtc-config` issues TURN credentials publicly

The endpoint can issue short-lived TURN credentials without authentication. Even with a short TTL, this can create cost/abuse risk.

Expected remediation:

- require resident/device auth for TURN credentials
- use client role from verified auth, not request headers alone
- add rate limits and monitor TURN allocation usage

### 7. Dependency audit is not production-clean

Audit findings observed during review:

- signaling server: 16 production findings, including critical `protobufjs`, high XML/parser issues, `node-forge`, and vulnerable `ws`
- Home app: 35 production findings, including 2 critical
- Intercom app: 30 production findings, including 1 critical
- CDK: 5 findings
- Lambda API: 2 moderate findings

Expected remediation:

- run safe audit fixes first
- plan breaking upgrades separately
- prioritize internet-facing server and Lambda packages before mobile build-time dependency chains
- document any accepted risk with package, advisory, reason, and revisit date

### 8. Infrastructure should reduce blast radius further

Observed hardening opportunities:

- SSH is open to the world; prefer SSM-only or restrict to an admin CIDR
- EC2 role includes broad S3 read-only access; replace with least-privilege bucket/object grants
- RDS should explicitly enable encryption/deletion protection/backups appropriate for production
- portal bucket should not be `DESTROY`/`autoDeleteObjects` in production
- add WAF/rate controls at ALB/CloudFront/API Gateway for public routes

### 9. coturn build source should be pinned

User data currently builds coturn from a cloned repository. Production builds should not depend on an unpinned default branch.

Expected remediation:

- pin a release tag or commit SHA
- verify source integrity where practical
- consider a trusted package/image instead of building from latest source during boot

### 10. Secret hygiene still needs final production discipline

No active server private key was found in source during review, and runtime secret loading already uses Secrets Manager. Remaining actions are mostly production hygiene.

Expected remediation:

- restrict Firebase mobile API keys by bundle ID, Android SHA fingerprints, and allowed APIs
- rotate production JWT, TURN shared secret, APNs auth key, and Firebase service-account material before production cutover
- remove or relocate APNs key metadata files if not operationally needed in the repo
- ensure generated build artifacts and native folders are not committed with secrets or stale configuration

## Suggested Acceptance Criteria

- unauthenticated WebSocket Home registration is rejected
- a resident can only register/watch/answer for apartments assigned to their verified account
- intercom JWT registration behavior continues to work
- invalid door-code responses never include the correct code
- `/healthz` returns a minimal 200 response and is used by the ALB
- `/debug/status` is unavailable without admin auth in production
- oversized JSON requests are rejected without high memory growth
- provisioning and RTC config endpoints are rate-limited and monitored
- `npm audit --omit=dev` is clean for signaling server and Lambda, or all remaining findings are documented
- CDK deploy preserves RDS and portal data in production
- SSH exposure is removed or restricted

## Notes

This step should be implemented in small slices because it touches live call flow.

Start with the low-risk leaks and public endpoint changes, then move to Home WebSocket authentication and call-control authorization with regression tests across Home app, Intercom app, and signaling server.
