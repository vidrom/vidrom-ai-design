# Vidrom Project Test Plan

## Purpose

This document defines the test strategy for the full Vidrom monorepo:

- `vidrom-ai-home` - resident mobile app
- `vidrom-ai-intercom` - intercom mobile app
- `vidrom-signaling-server` - signaling, REST APIs, push orchestration, and portal hosting
- `vidrom-cdk` - AWS infrastructure and deployment automation

The goal is to catch regressions in the highest-risk system behaviors before they reach production:

- incoming call delivery
- first-accept-wins reconciliation across multiple residents
- WebRTC session setup and cleanup
- watch-mode priority and preemption rules
- door-open and access-code flows
- resident, manager, admin, and device authentication boundaries
- manager scoping and portal CRUD isolation
- infrastructure, secret, and runtime configuration drift

This plan covers automated tests, manual validation, release gates, and production verification.

## Quality Goals

The project is considered healthy when it consistently proves these invariants:

1. A ring reaches the correct apartment and building only.
2. Only one resident can win an accept race for a call.
3. Non-winning resident devices stop ringing quickly through WebSocket and push fallback paths.
4. A watch session never overrides an active call, and a call always preempts a watch.
5. The home app remains the SDP offerer for call and watch sessions.
6. Call, watch, hangup, timeout, and door-open cleanup leave no stale active session state.
7. Admin and management APIs enforce server-side auth and building scoping.
8. Intercom device auth and resident auth are trusted only from verified server-side identity.
9. Runtime secrets, TURN config, TLS endpoints, and AWS networking stay aligned with the deployed topology.

## Scope

### In Scope

- Unit, component, integration, contract, and end-to-end behavior for all four packages
- Native mobile behavior that affects call delivery, push handling, and WebRTC setup
- REST APIs for resident, intercom, admin, and management consumers
- WebSocket signaling and push fanout behavior
- Lambda and EC2 deployment paths
- Database-backed business rules and ownership boundaries
- Security, observability, and operational recovery verification

### Out Of Scope

- Store listing compliance testing
- Formal load testing at internet-scale traffic volumes
- Penetration testing by an external security vendor

These items can be added later, but they are not release blockers for the current phase.

## Test Levels

### 1. Static Verification

Run on every branch and before merging:

```sh
npm run lint
```

This is the cheapest way to catch syntax, import, and obvious correctness issues across the monorepo.

### 2. Package-Level Automated Tests

Run on every branch and before merging:

```sh
cd vidrom-ai-home && npm test
cd vidrom-ai-intercom && npm test
cd vidrom-signaling-server && npm test
cd vidrom-cdk && npm test
```

Coverage gate for broad regression detection:

```sh
npm run test:coverage
```

Current automated coverage already exists in these areas:

- home app: call notification, CallKit, FCM, VoIP, config, Expo plugin, call manager
- intercom app: config helpers, call manager, WebRTC HTML bootstrap
- signaling server: auth, resident auth, WebSocket flow and ring behavior, Lambda auth, rate limits, portal CSP/security
- CDK: synthesized infrastructure assertions for RDS, VPC, Lambda, and security groups

### 3. Cross-Package Integration Tests

These tests should validate contracts between packages without relying on manual device interaction.

Priority integration coverage:

1. Home app and signaling server agree on canonical message names and payload shapes.
2. Intercom app and signaling server agree on ring, answer, candidate, watch, and watch-end flows.
3. Server REST auth matches mobile and portal token expectations.
4. Runtime ICE config returned by `/api/rtc-config` is consumable by both mobile apps.
5. Portal frontends remain compatible with the current Lambda and CSP responses.

Where possible, keep these as server-driven harness tests with mocked clients rather than full emulator tests.

### 4. Manual End-to-End Validation

Because the system depends on native push, CallKit, Android notifications, and real WebRTC media, a manual end-to-end pass is required before release.

Required hardware for a full pass:

- 1 Android intercom device or emulator/device pair running the intercom app
- 1 iOS home device
- 1 Android home device
- optional second home device for multi-resident race scenarios

### 5. Post-Deploy Production Verification

After infrastructure or runtime deploys, run a focused live check against the deployed environment.

This is especially important because the current deployment model is split:

- `vidrom-cdk` deploys infrastructure, Lambda bundles, and portal assets
- `vidrom-signaling-server` EC2 code is deployed separately

## Test Environments

### Local Developer Environment

Use for fast feedback and most automated coverage.

- home app Metro default: `8082`
- intercom app Metro default: `8081`
- signaling server local port: `8080`

### Shared Cloud Environment

Use for portal, secrets, TLS, push, and device-to-device validation.

- `https://signaling.vidrom.com`
- `wss://signaling.vidrom.com`

### Environment Risks

- AWS account restrictions can currently block EC2-mutating infra verification even when local code and tests are correct.
- iOS incoming-call behavior cannot be trusted from simulator-only testing.
- FCM and VoIP delivery must be validated on real devices.

## Test Data And Fixtures

Maintain a stable validation dataset with:

- at least 2 buildings
- at least 2 apartments per building
- at least 2 residents in one apartment for accept-race validation
- 1 manager assigned to only one building
- 1 admin user
- 1 provisioned intercom per building
- valid FCM and VoIP tokens for resident devices

Recommended fixture scenarios:

1. Single-resident apartment
2. Multi-resident apartment
3. Resident belonging to multiple apartments
4. Manager assigned to one building only
5. Revoked intercom device
6. Sleep-mode-enabled resident

## Feature Test Matrix

### Home App

Core coverage:

1. Login and identity restoration for Google and Apple flows
2. FCM token registration and refresh
3. VoIP token registration and refresh on iOS
4. Incoming ring UI in foreground, background, and relaunched states
5. Accept, decline, call-taken, timeout, and hangup handling
6. Offer creation, ICE exchange, and media state transitions
7. Door-open during active call and outside a call
8. Watch start, watch end, and cleanup
9. Sleep mode behavior and notification suppression rules
10. Runtime ICE config refresh before TURN expiry

Manual platform matrix:

- Android home app on real device
- iOS home app on real device

### Intercom App

Core coverage:

1. Provisioning and device authentication
2. Building configuration fetch and refresh
3. Apartment directory rendering
4. Ring initiation to target apartment
5. WebView bootstrap for WebRTC HTML
6. Answer creation and ICE relay
7. Watch-mode background media behavior without navigating to call UI
8. Cleanup when watch is preempted by a call
9. Door-code validation and door-open actions
10. Device revocation behavior and reconnect handling

Primary platform:

- Android device or emulator matching intercom hardware assumptions

### Signaling Server

Core coverage:

1. WebSocket registration for home and intercom roles
2. Apartment-to-building resolution
3. Building isolation across concurrent activity
4. Pending ring persistence and resend behavior
5. First-accept-wins conflict resolution
6. `call-taken` fanout over WebSocket, FCM, and VoIP paths
7. Timeout, unanswered, decline, and hangup resolution
8. Watch-mode arbitration rules
9. Door-open and audit-log persistence
10. `/api/rtc-config` secret-backed TURN response and cache headers
11. Resident auth, operator auth, and device auth enforcement
12. Rate limiting, CSP, and security headers for Lambda and portal routes

### Admin Portal

Core coverage:

1. Admin token verification and role enforcement
2. Full CRUD for buildings, apartments, users, devices, and notifications
3. Manager assignment and resident assignment flows
4. Audit-log filtering
5. Global settings edits
6. Portal rendering under current CSP and asset hosting rules

### Management Portal

Core coverage:

1. Manager token verification and subject binding
2. Building-scoped reads and writes only
3. Forbidden access to unassigned building data
4. Allowed apartment, resident assignment, device, and notification operations inside scope
5. Forbidden global settings and user-management actions
6. Portal rendering under current CSP and asset hosting rules

### CDK And Infrastructure

Core coverage:

1. VPC, NAT, RDS subnet, and security group synthesis assertions
2. Lambda placement and env wiring
3. TLS, DNS, and public endpoint assumptions
4. Secrets Manager references and runtime env propagation
5. Deployment script correctness for infra vs EC2 app split
6. Recovery steps for CloudFormation rollback scenarios

## Critical End-To-End Scenarios

These are the minimum release-blocking scenarios.

### Call Delivery

1. Intercom rings a single-resident apartment and the home app shows incoming UI.
2. Intercom rings a multi-resident apartment and all eligible devices ring.
3. One resident accepts and all other devices receive `call-taken` quickly.
4. All residents decline and the intercom receives the decline outcome.
5. Nobody accepts and the call resolves to unanswered after timeout.

### Media Negotiation

1. Home creates the offer and intercom answers.
2. ICE candidates exchange both ways and media establishes.
3. A transient disconnect produces deterministic cleanup.
4. Hangup from either side ends the session cleanly on both sides.

### Watch Mode

1. Resident starts watch when no session is active.
2. Second resident starts watch and replaces the first watcher.
3. Intercom starts a ring while watch is active and watch ends first.
4. Resident cannot start watch during an active call.

### Door Access

1. Resident opens the door from the home app outside a call.
2. Resident opens the door during a call.
3. Visitor enters a valid code and the door opens.
4. Visitor enters an invalid code and access is denied.

### Auth And Authorization

1. Resident endpoints reject forged or mismatched identity.
2. Intercom endpoints reject revoked or malformed device JWTs.
3. Admin portal rejects non-admin operators.
4. Management portal rejects out-of-scope building access.
5. Subject-based auth binding remains stable after email changes.

### Reliability And Recovery

1. Pending ring survives the home app reconnect path within TTL.
2. Token re-registration tolerates app restart.
3. Server restart behavior is understood and verified for in-memory token limitations.
4. `/debug/status` and operational logs provide enough state to diagnose delivery failures.

## Non-Functional Testing

### Security

Run on every release candidate:

1. Verify auth is always resolved server-side.
2. Verify manager queries are scoped by `building_managers`.
3. Verify resident ownership checks on apartment and call actions.
4. Verify portal CSP, CORS, and security headers.
5. Verify secrets are not required from checked-out fallback files in production.

### Performance

Lightweight, targeted checks are sufficient for now:

1. Ring-to-UI time on Android and iOS should be measured during manual end-to-end validation.
2. Offer-to-media-establishment time should be recorded during release testing.
3. Portal API list endpoints should be checked against realistic data volumes.

### Resilience

1. Verify behavior when the home client reconnects during a pending ring.
2. Verify behavior when the intercom disconnects mid-call.
3. Verify behavior when TURN credentials expire and the client refreshes ICE config.
4. Verify deployment rollback and recovery instructions remain usable.

## Recommended Execution Cadence

### Per Pull Request

Run:

```sh
npm run lint
npm run test:coverage
```

Expectation:

- all package tests pass
- no new lint failures
- changed area includes targeted regression coverage

### Before Merging Cross-Flow Changes

Required when changing call, push, auth, or contract behavior:

1. Re-run the affected package tests directly.
2. Run at least one local cross-package manual scenario.
3. If signaling contracts changed, verify all three sides: home app, intercom app, server.

### Before Release

Required sign-off:

1. Full lint and coverage suite green
2. Release-blocking end-to-end scenarios completed on real devices
3. Portal auth and scoping checks completed
4. Infra diff reviewed if `vidrom-cdk` changed
5. EC2 deploy script path reviewed if `vidrom-signaling-server` changed

### After Deploy

Required live checks:

1. `/api/rtc-config` returns expected no-store behavior and valid ICE config
2. admin and management portals load and authenticate
3. one live incoming call succeeds end to end
4. one watch session succeeds end to end
5. debug and audit trails reflect the live actions

## Entry And Exit Criteria

### Entry Criteria For Release Testing

1. The branch is code complete.
2. All touched packages have passing automated tests.
3. Required secrets and device credentials are available.
4. Test devices can reach the active signaling environment.

### Exit Criteria For Release Approval

1. No P0 or P1 failures remain open.
2. All release-blocking scenarios pass.
3. Auth, scoping, and call delivery regressions are ruled out.
4. Any AWS-blocked verification gaps are explicitly documented before shipping.

## Current Gaps To Close Over Time

These are not reasons to delay writing the plan, but they should guide future test investment:

1. More explicit automated cross-package contract tests for signaling payloads
2. Broader automated coverage for portal CRUD edge cases
3. A repeatable seeded environment for multi-building and multi-resident scenarios
4. Lightweight scripted end-to-end smoke tests for deployed environments
5. Better persistence and auditability for push-token and push-delivery diagnostics

## Ownership

- Mobile flow regressions: `vidrom-ai-home` and `vidrom-ai-intercom`
- Real-time and API contract regressions: `vidrom-signaling-server`
- Infra, secrets, and deployment regressions: `vidrom-cdk`
- Final release sign-off: validate the affected package plus every cross-package flow touched by the change

## Canonical Commands

```sh
# workspace
npm run lint
npm run test:coverage

# home app
cd vidrom-ai-home && npm test
cd vidrom-ai-home && npm run run-metro
cd vidrom-ai-home && npm run run-ios
cd vidrom-ai-home && npm run run-android

# intercom app
cd vidrom-ai-intercom && npm test
cd vidrom-ai-intercom && npm run run-metro
cd vidrom-ai-intercom && npm run run-android

# signaling server
cd vidrom-signaling-server && npm test
cd vidrom-signaling-server && npm start

# infrastructure
cd vidrom-cdk && npm test
cd vidrom-cdk && npm run build
cd vidrom-cdk && npx cdk diff
```

This test plan should be updated whenever a new user-facing flow, auth boundary, deployment path, or package-level test suite is added.