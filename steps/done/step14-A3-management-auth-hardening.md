# Step 14-A3 — Management Auth Hardening

## Problem

Management portal auth still starts from an email-based user lookup before loading scoped buildings.

## Dependencies

Requires [step14-A1-operator-subject-foundation.md](step14-A1-operator-subject-foundation.md).

## What To Do

For `/api/management/*` routes:

1. require a valid Google bearer token
2. resolve the manager by persisted Google subject first
3. use verified email fallback only for one-time subject migration
4. enforce `role = 'manager'` after trusted user resolution
5. keep building scope derived from `building_managers`
6. reject conflicting subject bindings

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/lambda/adminAuth.js` | Harden management auth lookup |
| `vidrom-signaling-server/lambda/handler.js` | Keep `/api/management/*` using the hardened auth helper |
| management auth tests | Add manager auth and scoping coverage |

## Verification

- [ ] management routes return `401` without valid bearer auth
- [ ] management lookup prefers stored Google subject over email
- [ ] manager building scoping still works correctly
- [ ] conflicting subject bindings are rejected