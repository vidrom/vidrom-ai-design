# Step 13 — Real Device Validation

## Purpose

Step 13 server, app, migration, and production auth-enforcement work is complete. The remaining Step 13 work requires a legitimate resident account on a real device so the positive production flow can be verified and `firebase_uid` backfill can be observed.

This file keeps those device-only checks separate from the completed production cutover record in [step13-final-production-cutover.md](../done/step13-final-production-cutover.md).

## Current Production State

Verified before this handoff:

1. resident users in production: `5`
2. apartment assignments in production: `4`
3. apartments with residents: `3`
4. resident users with `firebase_uid` already backfilled: `0`
5. unauthenticated resident endpoints return `401`
6. Step 13 migration and signaling server deploy are already complete in production

## Remaining Scope

Complete Step 13 only after a legitimate resident account is exercised successfully on a real device and the server-side identity binding is observed end to end.

## Device Validation Checklist

1. sign in to the home app with a legitimate resident account that already exists in production
2. verify apartment resolution succeeds without relying on a caller-supplied body email
3. verify FCM token registration succeeds for that resident
4. on iOS, verify VoIP token registration succeeds for that resident
5. place a real intercom-to-home call and verify the resident receives the incoming ring
6. verify delivery ack succeeds for the resident-owned device token
7. verify HTTP accept succeeds for the resident's apartment
8. verify an out-of-scope resident cannot ack or accept for a different apartment or call
9. verify the matching `users` row now has a non-null `firebase_uid`
10. verify later authenticated resident requests resolve by `firebase_uid` instead of email fallback

## Completion Trigger

Move this file to `done/` and mark Step 13 complete only after the positive resident production flow and `firebase_uid` backfill are both verified on real devices.