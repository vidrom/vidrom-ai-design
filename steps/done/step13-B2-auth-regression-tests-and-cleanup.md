# Step 13-B2 — Auth Regression Tests And Cleanup

## Problem

The auth hardening is security-sensitive. Without regression coverage, later refactors can easily reintroduce body-driven trust or leave compatibility bypasses behind.

## Dependencies

Best after:

- [step13-A2-authenticated-apartment-resolution.md](step13-A2-authenticated-apartment-resolution.md)
- [step13-A3-authenticated-token-registration.md](step13-A3-authenticated-token-registration.md)
- [step13-A4-authenticated-call-actions.md](step13-A4-authenticated-call-actions.md)
- [step13-B1-home-bearer-auth-transport.md](step13-B1-home-bearer-auth-transport.md)

## What To Do

### Tests

Add server coverage that proves:

1. unauthenticated resident endpoints return `401`
2. spoofed email does not affect apartment resolution
3. token registration cannot target another apartment
4. token registration cannot delete another resident's rows
5. call ack is rejected for unrelated apartment scope
6. HTTP accept is rejected for unrelated apartment scope
7. legitimate resident flows still work end-to-end

### Cleanup

After the new auth model is proven:

1. remove compatibility branches that still trust `email`, `userId`, or `apartmentId` from resident request bodies
2. remove any resident fallback that uses legacy role-based token ownership instead of bearer auth
3. keep only the identity paths that remain intentionally supported

## Files Touched

| File | Change |
|------|--------|
| server test files | Add auth and ownership regression coverage |
| `vidrom-signaling-server/src/httpRoutes.js` | Remove dead compatibility branches after tests pass |

## Verification

- [ ] server tests cover resident auth and ownership boundaries
- [ ] no resident endpoint still trusts body-provided identity for authorization
- [ ] legacy compatibility paths removed here do not break intended supported flows