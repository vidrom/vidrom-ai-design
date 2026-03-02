# Future Task — Persistent Device Storage

## Problem

Device registrations (created via the admin web UI) are stored in an in-memory `Map` in `vidrom-signaling-server/src/devices.js`. When the signaling server restarts (deploy, crash, EC2 reboot), all device data is lost. This means:

- Every intercom must be re-provisioned after a server restart
- Intercom JWTs reference `deviceId`s that no longer exist, causing "Device revoked or not found" errors
- Device auth was temporarily disabled in `wsHandler.js` as a workaround

## Current Architecture

- `devices.js` uses two in-memory Maps:
  - `devices: Map<deviceId, { deviceId, buildingId, name, status, createdAt }>`
  - `provisioningCodes: Map<code, deviceId>`
- Status lifecycle: `pending` → `active` → `revoked`
- Functions: `createDevice`, `validateProvisioningCode`, `getDevice`, `revokeDevice`, `listDevices`

## What Needs to Change

### Option A — DynamoDB (recommended for AWS)

1. Create a DynamoDB table via CDK (`vidrom-cdk-stack.ts`):
   - Table name: `vidrom-devices`
   - Partition key: `deviceId` (String)
   - GSI on `provisioningCode` for code lookups
   - TTL on provisioning codes (e.g., 24h expiry)

2. Add IAM permissions to the EC2 role for DynamoDB access

3. Replace `devices.js` in-memory Maps with DynamoDB calls:
   - `createDevice` → `PutItem`
   - `validateProvisioningCode` → `Query` on GSI, then `UpdateItem` to set status=active
   - `getDevice` → `GetItem`
   - `revokeDevice` → `UpdateItem` to set status=revoked
   - `listDevices` → `Scan` (acceptable for small device counts)

4. Add `@aws-sdk/client-dynamodb` and `@aws-sdk/lib-dynamodb` to signaling server dependencies

### Option B — SQLite (simpler, single-instance)

1. Install `better-sqlite3` in the signaling server
2. Create a SQLite database file at `/opt/vidrom-signaling/devices.db`
3. Replace in-memory Maps with SQL queries
4. Database file persists across server restarts (but not across EC2 replacements unless backed up)

### Option C — JSON file (simplest, least robust)

1. Write devices Map to a JSON file on disk after each mutation
2. Load from file on server startup
3. Simple but no concurrency protection; fine for low-volume use

## After Persisting Devices

1. **Re-enable device authentication** in `wsHandler.js` — revert the TODO comment block back to the original auth checks (token required, device must exist and be active)
2. **Remove the workaround** that was added to bypass auth
3. Test the full flow: create device → provision intercom → restart server → intercom reconnects successfully

## Files to Modify

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/devices.js` | Replace in-memory Maps with persistent storage |
| `vidrom-signaling-server/src/wsHandler.js` | Re-enable device auth checks |
| `vidrom-signaling-server/package.json` | Add DB client dependency |
| `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Add DynamoDB table + IAM (if Option A) |
