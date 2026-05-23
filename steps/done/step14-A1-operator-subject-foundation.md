# Step 14-A1 — Operator Subject Foundation

## Problem

Admin and management portal auth currently bind verified Google tokens to database users by `users.email`. That leaves operator authorization anchored to a mutable email claim instead of a stable Google identity.

## Dependencies

None.

This is the foundation for the rest of Step 14.

## What To Do

### Database

Add a nullable operator identity column on `users`, for example:

1. `google_subject`

Add a unique partial index for non-null values.

### Auth helper

In `vidrom-signaling-server/lambda/adminAuth.js`:

1. keep Google ID token verification with `google-auth-library`
2. expose a helper that reads the stable Google subject from `payload.sub`
3. resolve operator users by stored subject first
4. if no subject match exists, allow a verified-email fallback only for unbound rows
5. backfill the Google subject after a successful fallback
6. reject conflicting subject bindings

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/sql/` | Add operator subject migration |
| `vidrom-signaling-server/lambda/adminAuth.js` | Add subject-first auth resolution |
| `vidrom-ai-design/business-logic/database-schema.md` | Document the new operator identity field |

## Verification

- [ ] valid Google token resolves operator user by stored subject
- [ ] verified-email fallback backfills the subject for unbound rows
- [ ] conflicting subject bindings are rejected
- [ ] no route behavior changes yet beyond stronger identity binding