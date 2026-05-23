# Step 14-B1 — Portal Auth Regression Tests And Cleanup

## Problem

Portal auth is security-sensitive and easy to regress if tests do not pin down subject binding behavior.

## Dependencies

Best after:

1. [step14-A1-operator-subject-foundation.md](step14-A1-operator-subject-foundation.md)
2. [step14-A2-admin-auth-hardening.md](step14-A2-admin-auth-hardening.md)
3. [step14-A3-management-auth-hardening.md](step14-A3-management-auth-hardening.md)

## What To Do

### Tests

Add coverage that proves:

1. missing or invalid operator bearer tokens return `401`
2. stored Google subject is preferred over email lookup
3. verified-email fallback only works for unbound rows
4. conflicting subject bindings are rejected
5. admin role enforcement still works
6. management role and building scoping still work

### Cleanup

After the new auth model is proven:

1. remove dead email-primary assumptions from Lambda auth helpers
2. update portal docs to describe subject-first identity binding
3. document the production cutover steps in [step14-final-production-cutover.md](step14-final-production-cutover.md) for the operator subject migration

## Files Touched

| File | Change |
|------|--------|
| Lambda auth tests | Add operator auth regression coverage |
| `vidrom-signaling-server/lambda/adminAuth.js` | Remove dead compatibility assumptions if any remain |
| `vidrom-ai-design/admin-portal.md` | Update auth notes |
| `vidrom-ai-design/management-portal.md` | Update auth notes |

## Verification

- [ ] portal auth tests cover subject binding and scoping boundaries
- [ ] no operator auth path still depends on email as the primary identity key
- [ ] docs reflect the hardened auth model