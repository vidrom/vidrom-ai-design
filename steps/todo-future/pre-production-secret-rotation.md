# Future Task — Pre-Production Secret Rotation and Final Secret Audit

## Problem

Step 12-A2 removed hardcoded server-side secrets from the active runtime path and moved runtime secret loading to AWS Secrets Manager. That fixes the architecture, but it does not change the fact that some credentials previously existed in the workspace during development.

Before Vidrom goes live in production, all sensitive credentials that were ever committed, copied, or used in local development need to be rotated and re-issued so production starts from a clean trust boundary.

This is a pre-launch hardening task, not an immediate development blocker.

## What Must Be Rotated Before Production

### 1. Firebase Admin Service Account Key

The server-side Firebase admin credential must be replaced with a newly issued key.

**Why:**
- A service-account private key was present locally during development
- Firebase admin keys grant server-side access and should be treated as exposed once committed or copied into a repo/workspace

**Required actions:**
- Create a new Firebase service-account key for the signaling server
- Update the Secrets Manager secret used by the signaling server runtime
- Delete the old Firebase service-account key from Google Cloud / Firebase Console

### 2. APNs Auth Key

The APNs auth key used for VoIP pushes should be rotated if the current `.p8` key was ever committed or shared beyond a tightly controlled local environment.

**Why:**
- APNs auth keys are long-lived signing keys
- If a real key was committed, assume it may have been exposed

**Required actions:**
- Generate a new APNs auth key in Apple Developer
- Update the runtime secret metadata if `APN_KEY_ID` changes
- Update the Secrets Manager secret storing the `.p8` contents
- Revoke the old APNs auth key in Apple Developer

### 3. Twilio NTS Credentials

The Twilio credentials used by `/api/rtc-config` should be rotated before production if the current values were ever used in development, shared broadly, or stored outside the intended secret path.

**Why:**
- the signaling server now issues short-lived ICE config by calling Twilio NTS directly
- production should start with Twilio credentials that were never part of an earlier dev or transition flow
- rotating the auth token reduces the chance that prior testing credentials continue to grant relay access

**Required actions:**
- create or verify the production Twilio subaccount intended for Vidrom signaling
- rotate the Twilio auth token for that account, or move to a fresh subaccount if preferred
- update the runtime Secrets Manager JSON secret with `TWILIO_ACCOUNT_SID` and `TWILIO_AUTH_TOKEN`
- optionally set `TWILIO_NTS_TTL_SECONDS` if production wants a non-default ICE token TTL
- redeploy the signaling server so the new values are loaded

### 4. JWT Secret

The JWT secret used for device authentication must be production-specific and freshly generated.

**Why:**
- Device/auth tokens should be signed with a secret that has never been present in development notes, local testing, or transitional environments

**Required actions:**
- Generate a new strong random JWT secret
- Update the runtime Secrets Manager JSON secret
- Redeploy the signaling server

## What Does Not Need Rotation

### Firebase Mobile App Config Files

The following files are expected client-side Firebase app configuration and may remain committed if they are the intended production project values:

- `vidrom-ai-home/google-services.json`
- `vidrom-ai-home/GoogleService-Info.plist`
- `vidrom-ai-intercom/google-services.json`

These are not server-admin credentials. They identify the Firebase project to the client apps and are commonly shipped in mobile apps.

## Required AWS Updates

After rotating credentials, update these Secrets Manager entries:

### `vidrom/signaling/runtime`

Expected JSON fields:

```json
{
  "JWT_SECRET": "<new-random-jwt-secret>",
  "TWILIO_ACCOUNT_SID": "<twilio-account-sid>",
  "TWILIO_AUTH_TOKEN": "<twilio-auth-token>",
  "TWILIO_NTS_TTL_SECONDS": "86400",
  "APN_KEY_ID": "<apns-key-id>",
  "APN_TEAM_ID": "<apple-team-id>",
  "APN_BUNDLE_ID": "com.vidrom.ai.home",
  "APN_PRODUCTION": "true"
}
```

### `vidrom/signaling/firebase-service-account`

- Replace with the new Firebase service-account JSON

### `vidrom/signaling/apns-auth-key`

- Replace with the new APNs `.p8` file contents

## Deployment Sequence

Once the AWS account is unblocked and the new secrets are ready:

1. Update the three Secrets Manager entries with rotated values
2. Run `vidrom-cdk/cdk-deploy.sh`
3. Run `vidrom-cdk/deploy-server-ssm.sh`
4. Confirm the signaling server starts successfully with the new secret values
5. Confirm `/api/rtc-config` returns Twilio-backed ICE config for authenticated clients
6. Test VoIP push delivery on iOS
7. Test a full intercom-to-home call flow

## Final Secret Audit Before Launch

Before production release, run a final audit across the workspace and deployment configuration.

**Checklist:**
- No committed `service-account.json` remains in any repo
- No committed `.p8` APNs auth key remains in any repo
- No tracked backup config files like `google-services.json.bak` remain
- No hardcoded JWT, DB, or Twilio secrets remain in source or docs
- Runtime values come only from AWS Secrets Manager
- Local build artifacts containing obsolete secrets are deleted
- Rotated credentials are confirmed active and old credentials are revoked

## Exit Criteria

This task is complete when:

- All production credentials have been rotated
- Old credentials have been revoked
- AWS Secrets Manager holds only the rotated values
- The signaling server and portal APIs run successfully with the rotated secrets
- A final workspace scan shows no committed server-side secrets