# Step 13-A1 — Resident Auth Foundation

## Problem

The signaling server currently has no dedicated authentication layer for resident-facing HTTP endpoints. As a result, multiple endpoints rely on caller-supplied `email`, `userId`, and `apartmentId` fields as if they were trusted identity.

## Dependencies

None.

This is the foundation for every later Step 13 item.

## What To Do

### Server

In the signaling server:

1. Add a resident auth helper for `Authorization: Bearer <token>`.
2. Verify Firebase ID tokens with Firebase Admin.
3. Resolve a server-trusted resident context from PostgreSQL.
4. Expose a helper that returns fields such as:
   - Firebase UID
   - email
   - userId
   - apartmentIds
   - primaryApartmentId
   - buildingIds
5. Reject missing or invalid resident auth with `401`.
6. Reject valid auth with no resident mapping using a consistent `403` or `404` policy.

### Ownership model

Document and enforce the split clearly:

- intercom endpoints use device JWT
- portal endpoints use admin or management auth
- resident home endpoints use Firebase bearer auth

No endpoint should accept bearer auth and then trust a second conflicting identity from the body.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/auth.js` or a new helper | Add Firebase resident token verification and resident context resolution |
| `vidrom-signaling-server/src/httpRoutes.js` | Wire resident auth helper into resident-facing endpoints |

## Verification

- [ ] invalid bearer token returns `401`
- [ ] missing bearer token returns `401`
- [ ] valid resident bearer token resolves server-trusted resident context
- [ ] intercom JWT endpoints still work unchanged
- [ ] portal auth paths remain unaffected