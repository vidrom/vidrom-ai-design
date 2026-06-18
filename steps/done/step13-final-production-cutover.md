# Step 13 Final — Production Cutover After AWS Unlock

## Purpose

Step 13 code is implemented locally, but production is not finished until the database migration is applied, the signaling server code is deployed, and the resident flows are re-verified against the real AWS environment.

The production migration, deploy, and auth-enforcement portion of this step is now complete. The remaining real-device resident validation is tracked separately in [step13-real-device-validation.md](../todo/step13-real-device-validation.md).

## Current Status

Completed locally and in production:

1. resident HTTP endpoints require Firebase bearer auth
2. resident identity and apartment scope are derived server-side
3. token registration, delivery ack, and HTTP accept no longer trust body-provided resident identity
4. the home app sends bearer auth on resident HTTP requests
5. server regression tests for resident auth and ownership boundaries are passing
6. resident auth now prefers persisted `users.firebase_uid` and only falls back to verified email once in order to backfill the UID
7. the production database migration is applied
8. the EC2 signaling server code is deployed
9. unauthenticated production smoke checks pass

Still open outside this completed record:

1. verify the positive resident flow on real devices
2. verify `firebase_uid` backfill for at least one production resident

## AWS Production Verification — 2026-05-29

Completed after AWS access was restored:

1. `npm test` passed in `vidrom-signaling-server` (`43` tests passing).
2. The production database has the Step 13 migration:
  - `users.firebase_uid` exists
  - `uq_users_firebase_uid` exists
3. EC2 signaling server code was redeployed with `vidrom-cdk/deploy-server-ssm.sh`.
4. Post-deploy service checks passed:
  - `vidrom-signaling` is `active`
  - `coturn` is `active`
  - local `/healthz` returns `{"ok":true}`
  - public `https://signaling.vidrom.com/healthz` returns `{"ok":true}`
5. Production auth enforcement smoke checks passed:
  - unauthenticated `POST /api/home/resolve-apartment` returns `401`
  - unauthenticated `POST /register-fcm-token` returns `401`
  - unauthenticated `GET /api/rtc-config` returns `401`

Current production resident state at handoff to device validation:

1. resident users: `5`
2. apartment assignments: `4`
3. apartments with residents: `3`
4. resident users with `firebase_uid`: `0`

The remaining checks now require real-device execution and are tracked in [step13-real-device-validation.md](../todo/step13-real-device-validation.md).

Remaining live verification moved to [step13-real-device-validation.md](../todo/step13-real-device-validation.md).

## Files Already Landed

- `vidrom-signaling-server/src/auth.js`
- `vidrom-signaling-server/src/httpRoutes.js`
- `vidrom-signaling-server/sql/010-users-firebase-uid.sql`
- `vidrom-ai-home/authService.js`
- `vidrom-ai-home/HomeScreen.js`
- `vidrom-ai-home/fcmService.js`
- `vidrom-ai-home/voipService.js`
- `vidrom-ai-home/config.js`
- `vidrom-ai-home/useCallManager.js`
- `vidrom-signaling-server/test/auth.test.js`
- `vidrom-signaling-server/test/httpRoutes.residentAuth.test.js`

## Recorded Production Procedure

### 1. Reconfirm local state before shipping

From `vidrom-signaling-server`:

```sh
npm test
```

Only continue if the resident auth tests are still green.

### 2. Apply the DB migration

Apply:

```text
vidrom-signaling-server/sql/010-users-firebase-uid.sql
```

The migration adds:

1. nullable `users.firebase_uid`
2. a unique partial index on non-null Firebase UIDs

Expected post-migration state:

1. existing users keep working because `firebase_uid` starts null
2. the first valid resident bearer-auth request can backfill the UID from verified Firebase claims

### 3. Deploy only what Step 13 actually changed

This step does not require an infrastructure shape change.

Preferred deploy path:

1. deploy the EC2 signaling server code from `vidrom-cdk`
2. avoid a full `cdk deploy` unless there are separate infra changes to ship

Canonical code deploy:

```sh
cd vidrom-cdk
./deploy-server-ssm.sh
```

Only run a full stack deploy if there is another pending AWS change that actually needs CloudFormation:

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

### 4. Run production verification

The unauthenticated production smoke checks in this section are complete. The positive resident-device checks are tracked separately in [step13-real-device-validation.md](../todo/step13-real-device-validation.md).

Verify these resident flows against the live environment:

1. sign in to the home app with a legitimate resident account
2. apartment resolution succeeds without trusting a body email
3. FCM token registration succeeds
4. VoIP token registration succeeds on iOS
5. incoming ring still reaches the resident device
6. delivery ack still succeeds for the resident-owned token
7. HTTP accept still succeeds for the resident’s apartment
8. a resident outside the apartment scope cannot ack or accept the call

### 5. Verify Firebase UID backfill in the database

This remained incomplete at the time this production cutover record was moved to `done/`. Complete it through [step13-real-device-validation.md](../todo/step13-real-device-validation.md).

For at least one existing resident user who previously had `firebase_uid = NULL`, confirm that Step 13 backfilled the UID after a successful authenticated resident request.

Expected result:

1. the matching `users` row now has a non-null `firebase_uid`
2. future resident auth resolves by UID directly instead of email fallback

### 6. Re-check the exit criteria

Mark Step 13 done only when all are true in production:

1. resident-facing signaling HTTP endpoints require bearer auth
2. apartment resolution ignores spoofed email input
3. token registration cannot hijack another resident or apartment
4. call ack and HTTP accept reject unauthorized apartment access
5. the home app sends authenticated resident HTTP requests consistently
6. server tests cover auth and ownership boundaries
7. `firebase_uid` is persisted and being used as the primary resident identity lookup key

## Notes

### Why `firebase_uid` matters

Using verified email is better than trusting request bodies, but persisting Firebase UID is stricter because it binds the server-side resident record to the stable Firebase identity instead of a mutable email string.

### Why this should usually avoid `cdk deploy`

Step 13 changed application code and database schema, not the infrastructure topology. In the normal case, production completion should be:

1. DB migration
2. EC2 signaling code deploy
3. live verification

## Completion Note

The migration, EC2 deploy, and production auth-enforcement verification succeeded. This file is now a completed production cutover record; the only remaining Step 13 work is the real-device validation tracked in [step13-real-device-validation.md](../todo/step13-real-device-validation.md).