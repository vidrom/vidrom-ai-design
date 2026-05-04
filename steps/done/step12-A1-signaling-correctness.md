# Step 12-A1 — Signaling Correctness (12.1)

## Problem

The server's call-initiation path contains a confirmed ordering bug: the database insert for a new call uses `ringTimeoutSec` before that variable is declared and populated.

This can break the main ring flow before the call is fully recorded.

## Dependencies

None — this should land first.

## What To Do

### Server

In `vidrom-signaling-server/src/wsHandler.js`:

1. Resolve the building-level timeout before inserting the `calls` row.
2. Use the resolved timeout consistently for:
   - `calls.expires_at`
   - in-memory `pendingRing` lifetime
   - push TTL where relevant
3. Add a small helper for timeout resolution so the same logic is not split across the handler.

### Tests

Add a regression test that proves:

- a ring creates a DB call row successfully
- the timeout is applied consistently
- no reference error or undefined timeout is possible in the ring path

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/wsHandler.js` | Resolve ring timeout before call insert and reuse it consistently |
| server test file(s) | Add regression coverage for ring timeout resolution and DB insert |

## Verification

- [ ] Ring path inserts a call row without runtime failure
- [ ] Timeout comes from building config or global default as intended
- [ ] Ring expiry still works after the refactor