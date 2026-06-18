# Step 13 — Resident Auth Hardening

## Scope

Step 13 closes the resident-side trust gaps identified in the cross-repo review:

1. apartment resolution trusts caller-supplied email
2. push token registration trusts caller-supplied ownership fields
3. call ack and HTTP accept trust caller-supplied resident identity

This step keeps the existing product behavior, but changes the authority model so resident-facing signaling HTTP endpoints derive identity from verified bearer auth instead of request bodies.

## Goals

After Step 13:

1. resident HTTP endpoints require authenticated bearer tokens
2. the signaling server derives resident identity and apartment scope server-side
3. token registration cannot target another resident or apartment
4. call ack and HTTP accept cannot be forged for unrelated calls
5. the home app sends bearer auth consistently on resident HTTP requests

## Current Status

Local implementation is complete.

AWS production migration, deploy, and auth-enforcement checks are complete:

1. the Step 13 DB migration is applied (`users.firebase_uid` and `uq_users_firebase_uid` exist)
2. the EC2 signaling server was redeployed after AWS was unblocked
3. unauthenticated resident endpoints reject requests in production
4. production now has resident users and apartment assignments available for positive verification

Current production resident state before the final device validation pass:

1. resident users: `5`
2. apartment assignments: `4`
3. apartments with residents: `3`
4. resident users with `firebase_uid`: `0`

The remaining Step 13 work is positive resident-flow verification in [step13-real-device-validation.md](step13-real-device-validation.md).

## Implementation Order

Step 13 is now split into smaller sub-steps so the hardening can land incrementally.

| Order | File | Focus | Notes |
|---|---|---|---|
| 1 | [step13-A1-resident-auth-foundation.md](../done/step13-A1-resident-auth-foundation.md) | Add resident auth middleware and server-trusted resident context | Completed locally |
| 2 | [step13-A2-authenticated-apartment-resolution.md](../done/step13-A2-authenticated-apartment-resolution.md) | Replace body-driven apartment resolution with authenticated lookup | Completed locally |
| 3 | [step13-A3-authenticated-token-registration.md](../done/step13-A3-authenticated-token-registration.md) | Harden FCM and VoIP token registration ownership | Completed locally |
| 4 | [step13-A4-authenticated-call-actions.md](../done/step13-A4-authenticated-call-actions.md) | Harden delivery ack and HTTP accept ownership checks | Completed locally |
| 5 | [step13-B1-home-bearer-auth-transport.md](../done/step13-B1-home-bearer-auth-transport.md) | Add bearer auth transport in the home app | Completed locally |
| 6 | [step13-B2-auth-regression-tests-and-cleanup.md](../done/step13-B2-auth-regression-tests-and-cleanup.md) | Add auth regression coverage and remove compatibility fallbacks | Completed locally |
| 8 | [step13-final-production-cutover.md](../done/step13-final-production-cutover.md) | Record the completed production migration, deploy, and auth-enforcement verification | Completed |
| 9 | [step13-real-device-validation.md](step13-real-device-validation.md) | Finish positive resident production validation and verify `firebase_uid` backfill | Only open Step 13 work |

## Execution Notes

The intended sequence is:

1. add resident auth once on the server
2. move each resident endpoint to server-derived identity
3. update the home app to attach Firebase bearer tokens
4. lock in behavior with regression coverage
5. remove compatibility paths that still accept caller-owned identity fields
6. complete the remaining positive resident production checks on real devices and verify `firebase_uid` backfill

## Exit Criteria

- [x] resident-facing signaling HTTP endpoints require bearer auth
- [x] apartment resolution ignores spoofed email input at the server boundary
- [x] token registration cannot hijack another resident's token ownership
- [x] call ack and HTTP accept reject unauthorized apartment access
- [x] home app sends authenticated resident HTTP requests consistently
- [x] server tests cover auth and ownership boundaries
- [ ] positive resident production flow is verified on real devices
- [ ] `firebase_uid` backfill is observed and confirmed as the primary resident lookup path