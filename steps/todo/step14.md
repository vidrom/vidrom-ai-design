# Step 14 — Portal Auth Hardening

## Scope

Step 14 hardens the remaining operator-facing auth paths that still bind authorization to a mutable email string.

Today, the Lambda portal auth layer verifies a Google ID token correctly, but then resolves admin and management users from PostgreSQL by `users.email`.

That leaves two remaining trust gaps:

1. admin auth is still rebound by email instead of a stable Google subject
2. management auth and building scoping still start from email instead of a stable Google subject

This is weaker than the resident model completed in Step 13, which now prefers a persisted stable identity key and only falls back once for migration.

## Why This Improves Security

Yes, this is a real security improvement.

Using Google token verification plus role checks is already better than client-side trust, but using `email` as the database join key still leaves authorization tied to a mutable claim. Persisting the stable Google subject and resolving operators by that subject reduces the chance of accidental rebinding, conflicting records, or future auth regressions.

## Current Weak Points

The current email-bound auth path lives in:

1. `vidrom-signaling-server/lambda/adminAuth.js`
2. `vidrom-signaling-server/lambda/handler.js`

Affected route families:

1. `/api/admin/*`
2. `/api/management/*`

The current lookup pattern is effectively:

1. verify Google ID token
2. trust `payload.email`
3. load `users` row by `email + role`
4. load manager building scope from `building_managers`

The hardening target is:

1. verify Google ID token
2. trust `payload.sub` as the stable Google identity
3. resolve `users` row by persisted subject first
4. use verified email only as a one-time migration fallback when needed
5. persist the subject server-side so future requests do not depend on email lookup

## Goals

After Step 14:

1. admin auth resolves users by persisted stable Google subject instead of email
2. management auth resolves users by persisted stable Google subject instead of email
3. manager building scoping still derives from the server-trusted user record
4. operator auth cannot be rebound by changing only the email field in a token or DB row
5. portal auth regressions are covered by tests

## Proposed Data Model Change

Add a nullable Google subject column on `users`, for example:

1. `google_subject` or `google_sub`

Recommended shape:

1. nullable during migration
2. unique when non-null
3. initially backfilled lazily on first successful operator auth

This should mirror the resident-side migration pattern used for `firebase_uid` in Step 13.

## Implementation Plan

### A1. Persist Google subject on users

Add a DB migration that:

1. adds nullable `users.google_subject`
2. adds a unique partial index for non-null values

Update the schema docs accordingly.

### A2. Harden Lambda auth lookup

Update `vidrom-signaling-server/lambda/adminAuth.js` so that:

1. Google ID tokens are still verified with `google-auth-library`
2. admin and management users resolve by `google_subject = payload.sub` first
3. verified email fallback is only used when the DB row has no stored subject yet
4. successful fallback writes the Google subject back to the matching user row
5. conflicting existing subject bindings are rejected instead of overwritten

### A3. Preserve authorization semantics

Keep the current role and scoping rules intact:

1. admin still requires `role = 'admin'`
2. management still requires `role = 'manager'`
3. management scope still loads building IDs from `building_managers`
4. route authorization behavior should not widen during the auth migration

### A4. Add regression coverage

Add tests that prove:

1. valid Google token + matching stored subject authenticates
2. valid token + no stored subject falls back by verified email and backfills the subject
3. valid token + conflicting stored subject is rejected
4. admin routes still require admin role
5. management routes still require manager role and assigned building scope

## Files Likely Touched

### Server

1. `vidrom-signaling-server/lambda/adminAuth.js`
2. `vidrom-signaling-server/lambda/handler.js`
3. `vidrom-signaling-server/lambda/adminRoutes.js` if user CRUD needs to surface the new field
4. `vidrom-signaling-server/sql/` new migration file

### Design / docs

1. `vidrom-ai-design/business-logic/database-schema.md`
2. `vidrom-ai-design/admin-portal.md`
3. `vidrom-ai-design/management-portal.md`

### Tests

1. new Lambda auth tests under `vidrom-signaling-server/lambda/` or `vidrom-signaling-server/test/`

## Verification

- [ ] admin auth rejects missing or invalid bearer tokens
- [ ] management auth rejects missing or invalid bearer tokens
- [ ] stored Google subject is preferred over email lookup
- [ ] verified email fallback only works for unbound operator rows
- [ ] conflicting Google subject bindings are rejected
- [ ] manager building scoping still works after auth hardening
- [ ] regression tests cover admin and management auth binding rules

## Exit Criteria

- [ ] admin portal auth no longer depends on email as the primary DB identity key
- [ ] management portal auth no longer depends on email as the primary DB identity key
- [ ] a stable Google subject is persisted on operator user rows
- [ ] route authorization and management building scope remain correct
- [ ] production cutover plan is documented before shipping

## Notes

This should be treated as the operator-side counterpart to Step 13.

Step 13 hardened resident auth.
Step 14 should harden admin and management auth to the same standard.