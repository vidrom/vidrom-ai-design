# Step 12 — Real Device Validation

## Purpose

Step 12 server, infra, and documentation work is complete. The only remaining Step 12 work requires physical-device validation of the production signaling path and TURN relay behavior.

This file keeps those device-only checks separate from the completed AWS and repo work now recorded in:

1. [step12-A3-final-production-cutover.md](../done/step12-A3-final-production-cutover.md)
2. [step12-B1-db-and-turn-hardening.md](../done/step12-B1-db-and-turn-hardening.md)

## Remaining Scope

Complete Step 12 only after both of these are verified on real devices:

1. the home app and intercom app complete a real production call flow over `https://signaling.vidrom.com` and `wss://signaling.vidrom.com`
2. media still connects from a restrictive network path where TURN relay is likely required

## Current State

Already verified without real-device testing:

1. production signaling traffic is exposed through the secure TLS edge
2. `https://signaling.vidrom.com/healthz` is live
3. the production certificate is valid
4. the backend is not directly exposed on public port `8080`
5. the home app no longer relies on the production ATS insecure-load exception
6. production `/api/rtc-config` issues expiring TURN credentials instead of static credentials
7. the database is private and PostgreSQL ingress is security-group based only

## Device Validation Checklist

### A. Secure endpoint call flow

1. launch the home app against production and confirm it resolves the HTTP base URL to `https://signaling.vidrom.com`
2. launch the intercom app against production and confirm it resolves the WebSocket base URL to `wss://signaling.vidrom.com`
3. confirm neither app falls back to insecure `http://` or `ws://` transport in production
4. place a real intercom-to-home call and verify the signaling flow completes over the secure endpoints
5. on iOS, confirm the production app still works without a production ATS insecure transport exception

### B. Restrictive-network TURN relay

1. repeat a real intercom-to-home call while at least one side is on a restrictive network path where direct media is likely to fail
2. confirm the session still connects and media flows successfully
3. capture enough logs to confirm TURN relay was used when direct ICE candidates were not sufficient

## Completion Trigger

Move this file to `done/` and mark Step 12 complete only after the secure production call flow and restrictive-network TURN relay checks both pass on real devices.