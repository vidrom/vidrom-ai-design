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

## Implementation Order

Step 13 is now split into smaller sub-steps so the hardening can land incrementally.

| Order | File | Focus | Notes |
|---|---|---|---|
| 1 | [step13-A1-resident-auth-foundation.md](step13-A1-resident-auth-foundation.md) | Add resident auth middleware and server-trusted resident context | Foundation for every later step |
| 2 | [step13-A2-authenticated-apartment-resolution.md](step13-A2-authenticated-apartment-resolution.md) | Replace body-driven apartment resolution with authenticated lookup | Depends on A1 |
| 3 | [step13-A3-authenticated-token-registration.md](step13-A3-authenticated-token-registration.md) | Harden FCM and VoIP token registration ownership | Depends on A1 |
| 4 | [step13-A4-authenticated-call-actions.md](step13-A4-authenticated-call-actions.md) | Harden delivery ack and HTTP accept ownership checks | Depends on A1 |
| 5 | [step13-B1-home-bearer-auth-transport.md](step13-B1-home-bearer-auth-transport.md) | Add bearer auth transport in the home app | Best landed alongside A2-A4 |
| 6 | [step13-B2-auth-regression-tests-and-cleanup.md](step13-B2-auth-regression-tests-and-cleanup.md) | Add auth regression coverage and remove compatibility fallbacks | Best after A2-A4 and B1 |

## Execution Notes

The intended sequence is:

1. add resident auth once on the server
2. move each resident endpoint to server-derived identity
3. update the home app to attach Firebase bearer tokens
4. lock in behavior with regression coverage
5. remove compatibility paths that still accept caller-owned identity fields

## Exit Criteria

- [ ] resident-facing signaling HTTP endpoints require bearer auth
- [ ] apartment resolution ignores spoofed email input
- [ ] token registration cannot hijack another resident's token ownership
- [ ] call ack and HTTP accept reject unauthorized apartment access
- [ ] home app sends authenticated resident HTTP requests consistently
- [ ] server tests cover auth and ownership boundaries