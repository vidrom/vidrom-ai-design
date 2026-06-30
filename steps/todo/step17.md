# Step 17 — Automated Test Expansion

## Scope

Step 17 turns the current manual-heavy release validation plan into a repeatable automated regression system across the full Vidrom monorepo.

Nightly automation is intentionally not part of the immediate Step 17 delivery target. This step should establish the core pull-request and local regression layers first, while keeping the heavier nightly suites tracked as future follow-up work.

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

Initial Step 17 progress already landed:

- GitHub Actions now runs monorepo lint, coverage suites, and the CDK build on push and pull request events
- root workspace scripts now expose `npm run test`, `npm run test:coverage`, `npm run build:ci`, and `npm run ci` for a single entrypoint into the shared validation flow
- the Home app manifest and lockfile now include the Google Sign-In dependency that was already required by the app source and Expo plugin configuration, removing a real CI install/lint mismatch
- a shared signaling contract helper now defines canonical payload shapes for the core `ring`, `accept`, `decline`, `call-taken`, `offer`, `answer`, `candidate`, `watch`, `watch-end`, `hangup`, and `open-door` flows used by Home, Intercom, and signaling-server tests
- a reusable headless scenario-runner foundation now exists under `vidrom-signaling-server/test/e2e/`, with fake Home and Intercom clients driving the real signaling-server WebSocket handler through multi-step scenarios
- the headless scenario runner now covers late-join pending-ring replay, late-join `call-taken`, decline-all rejection, accepted-call door-open cleanup, second-watcher displacement, and watch rejection while the intercom is already on a call
- the expanded scenario-runner suite is now validated by the default root `npm run test:coverage` gate, and signaling-server `wsHandler` coverage increased as a result
- the headless scenario runner now also covers pending-ring timeout expiry, ringing-home disconnect-as-decline, non-winning home disconnect during an accepted call, and accepted-home disconnect ending the active call with `peer-disconnected` sent to the intercom
- after the latest scenario expansion, the signaling-server coverage gate now runs 57 passing tests and `wsHandler.js` coverage increased again to 67.86% statements
- the headless scenario runner now also asserts `call-taken` push fallback to offline apartment devices through fake APNs/FCM adapters, including stale-token cleanup on invalid-device responses
- after the latest push-fallback scenario expansion, the signaling-server coverage gate now runs 59 passing tests and `wsHandler.js` coverage increased again to 70.69% statements
- the headless scenario runner now also covers WebSocket accept reconciliation after a prior HTTP accept from the same resident, proving the WS session upgrades in-memory state without relaying a duplicate intercom `accept`
- after the latest HTTP-reconciliation scenario expansion, the signaling-server coverage gate now runs 60 passing tests with `wsHandler.js` holding at 70.69% statements
- the signaling server now includes a first PostgreSQL-backed integration harness under `test/integration/` that boots disposable Postgres 16 with Testcontainers, applies the real SQL schema files, and exercises resident-auth behavior against actual tables instead of mocked query rows
- the initial PostgreSQL-backed slice now covers resident scope resolution, resident `firebase_uid` backfill, admin `google_subject` backfill, manager building-scope loading, management-route building-scope enforcement and persistence for apartment create/update/delete, resident add/remove/list, device create/revoke/reprovision/list including persisted provisioning codes, notification create/delete/list, audit-log visibility, and delivery/device-health reporting, admin-route persistence for building and apartment listings plus CRUD, user create/update/delete/list, manager and resident assign/remove/list flows, intercom provisioning/update/delete/list and door-code hashing, notification CRUD, settings reads and updates, filtered audit-log and client-error queries, and system delivery/device-health summary views, Home token-registration ownership, persisted Home call `ack` and HTTP `accept` paths, HTTP accept-timeout reversion, stale push-token cleanup during real WebSocket ring delivery, stale push-token cleanup during `call-taken` fallback after first-accept-wins, stale push-token cleanup during HTTP accept `call-taken` fallback, unanswered ring expiry persistence, late-join ringing and call-taken audit persistence, decline-all rejection persistence, ringing-home disconnect rejection persistence, accepted-home disconnect end-of-call persistence, explicit home hangup plus door-open end-of-call persistence, max-call-duration timeout persistence, intercom hangup and intercom disconnect end-of-call persistence, sleep-mode ring skipping and partial delivery filtering, and retry-attempt plus ack-suppression, retry-time stale-token cleanup, and `delivery-degraded` persistence against the real schema, with the integration suite exposed through `npm run test:integration` and designed to skip cleanly when no container runtime is available
- the backend automation layer now also includes focused Lambda handler route-dispatch coverage for admin and management endpoints, including parsed JSON bodies, decoded setting keys, query-parameter forwarding, unknown-route 404s, and 500 fallback handling, so the portal entrypoint wiring is exercised in addition to the lower-level route modules and PostgreSQL-backed persistence slices

The main gaps are:

- the scenario runner now covers the main call/watch foundation plus late-join, timeout, disconnect-as-decline, core door-open/watch contention, `call-taken` push-fallback cleanup, and HTTP-accept reconciliation cases, but it still does not cover richer reconnect/resume semantics through the high-level runner
- database-backed behavior is still mostly mocked, but the first real PostgreSQL-backed auth, scope, Home token-registration, and core call ack/accept persistence slice now exists and can be expanded into broader retry cleanup, timeout reversion, and additional manager/admin route coverage
- portal CRUD/scoping behavior is not covered by browser automation
- WebRTC media setup is not smoke-tested with fake media in CI
- mobile UI automation is not yet present

For this step, the priority is to land the foundations that can run reliably on every pull request or on-demand locally. Nightly-only suites should remain documented but are not required for Step 17 completion.

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

## Step Boundary

Step 17 should be considered complete once the repo has strong pull-request automation, shared signaling contract checks, headless cross-package scenario coverage for critical call/watch flows, and an updated manual release checklist.

The following items are explicitly valuable but deferred beyond the main Step 17 completion gate:

- nightly portal browser runs
- nightly fake-media WebRTC smoke tests
- nightly Android emulator and Home/Intercom Maestro runs
- nightly deployed smoke checks against the shared cloud environment

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

### Future Nightly Expansion

These suites should be added after the core Step 17 pull-request automation is stable and useful:

- Playwright portal suite
- fake-media WebRTC smoke tests
- Android emulator Maestro smoke tests for Home and Intercom
- deployed smoke checks against the shared cloud environment if safe test data is available

### Before Release

- all Step 17 CI automation green
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
- emulator smoke tests can run locally and on demand without real APNs or FCM delivery, with nightly scheduling deferred to a future follow-up
- manual test plan is updated so real-device testing focuses on native OS and production-only risks

## Notes

Prioritize the headless cross-package scenario runner before mobile emulator automation. It will catch more Vidrom-specific regressions with less flakiness than device UI tests.

Mock APNs and FCM in CI. Assert outgoing push payloads and target selection in automated tests, then keep real push delivery checks as release smoke tests.

Use stable seeded test data with at least two buildings, two apartments per building, two residents in one apartment, one manager assigned to one building, one admin, one provisioned intercom per building, and one revoked intercom device.
