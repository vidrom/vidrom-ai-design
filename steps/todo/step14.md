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

## Current Status

Local implementation is complete.

Production migration, deploy, and live validation are complete:

1. the `google_subject` migration is present in production
2. the Lambda-backed portal stack was redeployed with Step 14 auth changes
3. unauthenticated admin and management API requests return `401`
4. production currently has `2` admin rows, `1` admin row with `google_subject`, and `1` manager row assigned to `BuildingOne` with `google_subject` backfilled after live sign-in
5. management live sign-in and building scoping were verified in production
6. one legacy admin row remains unbound and will backfill `google_subject` on its first successful sign-in, which is accepted as non-blocking because the deployed auth path already prefers subject first and uses verified email only as a one-time migration fallback

Step 14 is complete. The closeout record for the last live-validation pass is in [step14-live-portal-validation.md](../done/step14-live-portal-validation.md).

## Proposed Data Model Change

Add a nullable Google subject column on `users`, for example:

1. `google_subject` or `google_sub`

Recommended shape:

1. nullable during migration
2. unique when non-null
3. initially backfilled lazily on first successful operator auth

This should mirror the resident-side migration pattern used for `firebase_uid` in Step 13.

## Implementation Order

Step 14 is split so the operator auth migration can land incrementally.

| Order | File | Focus | Notes |
|---|---|---|---|
| 1 | [step14-A1-operator-subject-foundation.md](../done/step14-A1-operator-subject-foundation.md) | Add persisted Google subject and subject-first auth resolution | Completed locally |
| 2 | [step14-A2-admin-auth-hardening.md](../done/step14-A2-admin-auth-hardening.md) | Harden `/api/admin/*` auth binding | Completed locally |
| 3 | [step14-A3-management-auth-hardening.md](../done/step14-A3-management-auth-hardening.md) | Harden `/api/management/*` auth binding and keep building scope correct | Completed locally |
| 4 | [step14-B1-portal-auth-regression-tests-and-cleanup.md](../done/step14-B1-portal-auth-regression-tests-and-cleanup.md) | Add regression tests and remove email-primary leftovers | Completed locally |
| 5 | [step14-final-production-cutover.md](../done/step14-final-production-cutover.md) | Record the completed migration, Lambda deploy, and auth-enforcement smoke verification | Completed |
| 6 | [step14-live-portal-validation.md](../done/step14-live-portal-validation.md) | Record the final live validation pass and closure decision | Completed |

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

- [x] admin auth rejects missing or invalid bearer tokens
- [x] management auth rejects missing or invalid bearer tokens
- [x] stored Google subject is preferred over email lookup in the deployed code path
- [x] verified email fallback only works for unbound operator rows in the deployed code path
- [x] conflicting Google subject bindings are rejected by the deployed code path and tests
- [x] manager building scoping still works after auth hardening in live positive verification with the production manager account
- [x] regression tests cover admin and management auth binding rules

## Exit Criteria

- [x] admin portal auth no longer depends on email as the primary DB identity key in the deployed code path
- [x] management portal auth no longer depends on email as the primary DB identity key in the deployed code path
- [x] a stable Google subject is persisted on operator rows that have completed post-deploy sign-in; any remaining legacy row will backfill on first successful sign-in
- [x] route authorization and management building scope remain correct in live positive verification
- [x] production cutover plan is documented
- [x] live portal verification is complete for the production paths required to close this step

## Notes

This should be treated as the operator-side counterpart to Step 13.

Step 13 hardened resident auth.
Step 14 should harden admin and management auth to the same standard.

## Execution Notes

The intended sequence is:

1. persist a stable Google subject on operator users
2. harden admin auth to resolve by subject first
3. harden management auth to resolve by subject first while preserving building scoping
4. lock in the behavior with regression tests and doc cleanup
5. complete the live validation pass and closure record in [step14-live-portal-validation.md](../done/step14-live-portal-validation.md)