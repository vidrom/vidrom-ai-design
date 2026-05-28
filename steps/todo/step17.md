# Step 17 — Automated Test Expansion

## Scope

Step 17 turns the current manual-heavy release validation plan into a repeatable automated regression system across the full Vidrom monorepo.

The goal is not to eliminate all real-device testing. Native APNs, FCM wakeup, CallKit, Android full-screen notifications, real camera/microphone behavior, and final production connectivity still need focused device validation. The goal is to move most business-critical regression coverage into CI and headless test harnesses so manual testing becomes a small release smoke instead of the primary safety net.

## Goals

After Step 17:

1. every pull request runs lint, package tests, coverage, and CDK build checks in CI
2. signaling contracts are validated across Home app, Intercom app, and signaling server
3. core call, watch, door-open, auth, and cleanup scenarios can run headlessly without Android or iOS devices
4. server integration tests cover real PostgreSQL schema/query behavior for high-risk ownership and call flows
5. admin and management portal behavior is covered by browser automation
6. WebRTC signaling and fake-media offer/answer/candidate exchange can be smoke-tested in automation
7. mobile emulator smoke tests cover basic Home and Intercom UI flows without depending on real push delivery
8. real-device manual testing is reduced to native OS boundaries and final release verification

## Current Status

The repository already has useful package-level automated coverage:

- Home app tests cover CallKit, FCM, VoIP, call notifications, config helpers, Expo plugins, and `useCallManager`
- Intercom app tests cover config helpers, `useCallManager`, and WebRTC HTML bootstrap behavior
- signaling server tests cover auth, resident ownership, WebSocket ring/call flow, Lambda admin auth, rate limits, CSP/security headers, and startup config
- CDK tests cover synthesized infrastructure expectations

The main gaps are:

- CI currently runs lint only and does not run coverage, package tests, or CDK build checks
- there is no shared signaling message schema/contract gate across Home, Intercom, and server
- end-to-end call/watch/door behavior is mostly validated through package-level mocks and manual device testing rather than a cross-package scenario runner
- database-backed behavior is often mocked rather than verified against a real seeded PostgreSQL schema
- portal CRUD/scoping behavior is not covered by browser automation
- WebRTC media setup is not smoke-tested with fake media in CI
- mobile UI automation is not yet present

## Implementation Order

| Order | Focus | Primary Files / Area | Notes |
|---|---|---|---|
| 1 | Expand CI gates | `.github/workflows/`, root `package.json`, package scripts | Run `npm run lint`, `npm run test:coverage`, and `npm run build --prefix vidrom-cdk` on pull requests and pushes |
| 2 | Add shared signaling contracts | new shared schema module or test fixture area, Home/Intercom/server tests | Validate canonical payloads for `ring`, `accept`, `decline`, `call-taken`, `offer`, `answer`, `candidate`, `watch`, `watch-end`, `hangup`, and `open-door` |
| 3 | Build headless cross-package scenario runner | `vidrom-signaling-server/test/e2e/` or workspace-level `test/e2e/` | Run the signaling server in-process with fake Home and Intercom WebSocket clients, fake push adapters, and deterministic timers |
| 4 | Automate critical call scenarios | scenario runner tests | Cover single-resident ring, multi-resident ring, first-accept-wins, non-winner `call-taken`, decline-all, unanswered timeout, offer/answer relay, ICE relay, hangup cleanup, and reconnect pending-ring behavior |
| 5 | Automate watch and door scenarios | scenario runner tests | Cover watch start/end, second watcher replacement, call preempting watch, watch rejected during active call, resident door-open, call-time door-open, valid door code, and invalid door code |
| 6 | Add real PostgreSQL integration tests | signaling server test setup, SQL migrations/fixtures | Use Testcontainers or an equivalent local Postgres harness to verify building isolation, resident ownership, manager scope, device revocation, call persistence, token cleanup, and audit logs against real schema/query behavior |
| 7 | Add portal browser automation | Playwright tests for admin and management portals | Cover admin CRUD, manager building scope, forbidden cross-building access, auth rejection, CSP compatibility, and portal/API contract stability |
| 8 | Add fake-media WebRTC smoke tests | Playwright/Chromium or Node/browser harness | Use fake media devices to prove offer/answer/candidate exchange reaches a connected peer state without real devices |
| 9 | Add mobile emulator smoke tests | Maestro tests for Home and Intercom | Cover basic login/setup screens, apartment list/ring initiation, incoming-call UI trigger, accept/decline UI, watch start/stop UI, and door-code UI using test hooks or seeded APIs |
| 10 | Add deployed smoke scripts | signaling server scripts or workspace scripts | Verify `/healthz`, authenticated `/api/rtc-config`, portal load/auth, WSS connectivity, and a headless ring/accept/hangup against the deployed environment |
| 11 | Reframe manual release checklist | `vidrom-ai-design/test-plan.md` | Limit manual device testing to APNs/FCM wake, CallKit, Android full-screen notifications, real media, and final production sanity checks |

## Recommended Test Cadence

### Every Pull Request

- lint all packages
- run all package tests with coverage gates
- run CDK build and infrastructure tests
- run signaling contract tests
- run headless cross-package scenario tests for affected call/auth flows

### Every Pull Request Touching Calls, Auth, Push, WebRTC, Door Access, Or Portals

- run the full headless scenario suite
- run PostgreSQL integration tests
- run affected portal browser tests

### Nightly

- Playwright portal suite
- fake-media WebRTC smoke tests
- Android emulator Maestro smoke tests for Home and Intercom
- deployed smoke checks against the shared cloud environment if safe test data is available

### Before Release

- all CI and nightly automation green
- deployed smoke checks green
- minimal real-device pass for native boundaries:
  - iOS VoIP push wakes the app and shows CallKit
  - Android FCM/full-screen notification appears from background/relaunched states
  - real Home and Intercom devices establish media
  - one production watch session succeeds
  - one production door-open/code path succeeds

## Suggested Acceptance Criteria

- CI fails if lint, coverage, package tests, or CDK build fail
- CI runs all package-level tests on every pull request
- a shared contract test fails when any side changes a signaling message name or required payload shape incompatibly
- headless scenarios prove ring, accept, offer, answer, ICE, hangup, timeout, decline, watch, call preemption, and door-open behavior without mobile devices
- first-accept-wins is tested with at least two simulated Home clients in the same apartment
- non-winning residents receive `call-taken` through WebSocket and push-fallback assertions
- PostgreSQL integration tests use seeded multi-building, multi-apartment, multi-resident fixtures
- manager/admin portal browser tests prove scoped access and forbidden access boundaries
- fake-media WebRTC smoke proves a browser peer connection can be established through the signaling contract
- emulator smoke tests can run locally and in CI/nightly without real APNs or FCM delivery
- manual test plan is updated so real-device testing focuses on native OS and production-only risks

## Notes

Prioritize the headless cross-package scenario runner before mobile emulator automation. It will catch more Vidrom-specific regressions with less flakiness than device UI tests.

Mock APNs and FCM in CI. Assert outgoing push payloads and target selection in automated tests, then keep real push delivery checks as release smoke tests.

Use stable seeded test data with at least two buildings, two apartments per building, two residents in one apartment, one manager assigned to one building, one admin, one provisioned intercom per building, and one revoked intercom device.
