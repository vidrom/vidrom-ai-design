# Step 12-C1 — Critical Server And Infra Tests (12.6)

## Problem

The codebase has real production complexity but minimal automated protection.

The signaling server especially needs tests around state transitions, race behavior, and reconnect scenarios.

## Dependencies

A1 should land first for the initial ring-path regression case. The remaining coverage can be added incrementally as A2, A3, B1, and B2 land.

## What To Do

### Signaling server

Add tests for:

1. ring -> accept -> offer -> hangup
2. first-accept-wins across multiple residents
3. HTTP accept reconciliation with WS accept
4. ring expiry and unanswered transitions
5. watch start and watch end behavior
6. token cleanup on known-bad push errors

### CDK

1. Replace the placeholder SQS example test.
2. Add assertions for the actual resources and security posture you expect.

### Mobile

At minimum, add focused tests around config resolution and call state helpers where practical.

## Files Touched

| File | Change |
|------|--------|
| server test harness and test files | Add critical signaling flow coverage |
| `vidrom-cdk/test/vidrom-cdk.test.ts` | Replace scaffold test with real stack assertions |
| any extracted mobile utility modules | Add focused tests for config and call-state helpers |

## Verification

- [ ] Critical server state transitions are covered by automated tests
- [ ] CDK tests assert real stack behavior, not scaffolding defaults
- [ ] CI or local test scripts can run the new coverage predictably