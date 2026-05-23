# Step 14-A2 — Admin Auth Hardening

## Problem

Admin portal authorization still depends on looking up `users.role = 'admin'` by email after token verification.

## Dependencies

Requires [step14-A1-operator-subject-foundation.md](step14-A1-operator-subject-foundation.md).

## What To Do

For `/api/admin/*` routes:

1. require a valid Google bearer token
2. resolve the operator by persisted Google subject first
3. use verified email fallback only for one-time subject migration
4. enforce `role = 'admin'` only after trusted user resolution
5. return `401` for missing or invalid auth and reject conflicting subject bindings

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/lambda/adminAuth.js` | Harden admin auth lookup |
| `vidrom-signaling-server/lambda/handler.js` | Keep `/api/admin/*` using the hardened auth helper |
| admin auth tests | Add admin-specific auth coverage |

## Verification

- [ ] admin routes return `401` without valid bearer auth
- [ ] admin lookup prefers stored Google subject over email
- [ ] admin role enforcement still works correctly
- [ ] conflicting subject bindings are rejected