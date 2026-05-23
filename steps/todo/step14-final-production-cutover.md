# Step 14 Final — Production Cutover After AWS Unlock

## Purpose

Step 14 is not fully complete in production until the operator subject migration is applied, the Lambda-backed portal API is redeployed, and the hardened admin and management flows are re-verified against the live AWS environment.

Use this step only after the AWS account is unlocked again and the local Step 14 implementation work is already landed.

## Current Status

This step assumes these Step 14 items are already completed locally:

1. operator subject persistence and subject-first auth resolution are implemented
2. `/api/admin/*` auth resolves operators by stable Google subject first
3. `/api/management/*` auth resolves operators by stable Google subject first
4. portal auth regression tests for subject binding, role enforcement, and manager scoping are passing
5. portal auth docs are updated to describe subject-first identity binding

Not completed in AWS yet:

1. apply the Step 14 database migration that adds the operator Google subject column
2. deploy the updated Lambda/API stack so the hardened portal auth code is live
3. verify admin and management auth flows against the real AWS database and Google auth path

## Files Expected To Be Landed Before This Step Runs

- `vidrom-signaling-server/lambda/adminAuth.js`
- `vidrom-signaling-server/lambda/handler.js`
- `vidrom-signaling-server/lambda/adminRoutes.js` if Step 14 surfaced the new field there
- the Step 14 operator subject migration file under `vidrom-signaling-server/sql/`
- portal auth tests under `vidrom-signaling-server/test/` or `vidrom-signaling-server/lambda/`
- `vidrom-ai-design/admin-portal.md`
- `vidrom-ai-design/management-portal.md`
- `vidrom-ai-design/business-logic/database-schema.md`

## Do This When AWS Is Unlocked

### 1. Reconfirm local state before shipping

From `vidrom-signaling-server`:

```sh
npm test
```

Only continue if the Step 14 auth tests are still green.

### 2. Apply the DB migration

Apply the Step 14 SQL migration created during [step14-A1-operator-subject-foundation.md](../done/step14-A1-operator-subject-foundation.md).

Expected post-migration state:

1. `users` has a nullable operator Google subject column
2. the column is unique when non-null
3. existing operator rows keep working because the field starts null
4. the first successful operator auth request can backfill the subject for an unbound row

### 3. Deploy the Lambda-backed portal stack

Step 14 changes Lambda portal auth behavior, so production completion does require a CDK deploy.

Canonical deploy path:

```sh
cd vidrom-cdk
./cdk-deploy.sh
```

If CloudFormation is still stuck in `UPDATE_ROLLBACK_FAILED`, recover first:

```sh
aws cloudformation continue-update-rollback \
  --stack-name VidromSignalingStack \
  --resources-to-skip ApiLambda91D2282D \
  --region us-east-1
```

Then rerun:

```sh
cd vidrom-cdk
./cdk-deploy.sh
```

### 4. Run production verification for admin auth

Verify these admin flows against the live environment:

1. sign in to the admin portal with a legitimate admin account
2. admin API requests succeed with a valid Google bearer token
3. admin API requests fail with missing or invalid bearer auth
4. an admin row that previously had no stored subject is backfilled after successful auth
5. a non-admin operator cannot access admin-only routes

### 5. Run production verification for management auth

Verify these management flows against the live environment:

1. sign in to the management portal with a legitimate manager account
2. management API requests succeed with a valid Google bearer token
3. manager building scope still matches the operator's server-side assignments
4. a manager cannot access data outside the assigned building scope
5. missing or invalid bearer auth returns `401`

### 6. Verify Google subject backfill in the database

For at least one existing admin and one existing manager who previously had a null operator subject, confirm that Step 14 backfilled the subject after a successful authenticated operator request.

Expected result:

1. the matching `users` rows now have non-null stored Google subjects
2. future operator auth resolves by subject directly instead of email fallback

### 7. Re-check the exit criteria

Mark Step 14 done only when all are true in production:

1. admin portal auth no longer depends on email as the primary DB identity key
2. management portal auth no longer depends on email as the primary DB identity key
3. operator Google subject is persisted on admin and manager rows
4. admin role enforcement still works after the migration
5. manager building scoping still works after the migration
6. conflicting subject bindings are rejected
7. Step 14 regression tests are green and the live deploy is verified

## Notes

### Why Step 14 needs `cdk deploy`

Unlike Step 13, this work hardens the Lambda-backed portal auth path, so shipping the live change requires updating the stack artifact that carries the Lambda code.

### Why email fallback should remain temporary

Verified email is acceptable only as a one-time migration bridge for rows that do not yet have a stored subject. After backfill, operator auth should bind to the stable Google subject on every request.

## Completion Trigger

After the DB migration, CDK deploy, and live portal verification succeed, move Step 14 out of `todo` into the appropriate completed location and update the parent Step 14 checklist.