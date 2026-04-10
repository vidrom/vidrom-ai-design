# Step 11-B2 — Persist Ringing Sessions in DB (11.3)

## Problem

The current pending ring state lives entirely in memory (`pendingRings` Map in `connectionState.js`). If the signaling server restarts mid-ring, the ring is lost — the intercom thinks it's ringing, but the server has no record. Retries (B3) need durable state to know which devices to retry.

## Dependencies

Phase A (callId, acks). This is the foundation for B3 (retry orchestration).

## What To Do

### New table: `call_delivery_attempts`

Create migration `007-call-delivery-attempts.sql`:

```sql
CREATE TABLE IF NOT EXISTS call_delivery_attempts (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id           UUID NOT NULL REFERENCES calls(id) ON DELETE CASCADE,
    user_id           UUID REFERENCES users(id) ON DELETE SET NULL,
    device_token      TEXT NOT NULL,
    token_type        VARCHAR(10) NOT NULL CHECK (token_type IN ('fcm', 'voip')),
    platform          VARCHAR(10) NOT NULL CHECK (platform IN ('ios', 'android')),
    attempt_number    INTEGER NOT NULL DEFAULT 1,
    delivery_state    VARCHAR(30) NOT NULL DEFAULT 'queued'
                      CHECK (delivery_state IN (
                          'queued', 'push-sent', 'push-failed',
                          'push-received', 'app-awake', 'incoming-ui-shown',
                          'accepted', 'declined', 'timed-out',
                          'skipped-sleep-mode'
                      )),
    last_error        TEXT,
    last_attempt_at   TIMESTAMPTZ,
    acked_at          TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_call_delivery_attempts_call ON call_delivery_attempts (call_id);
CREATE INDEX idx_call_delivery_attempts_state ON call_delivery_attempts (call_id, delivery_state);
```

### Extend `calls` table

Add an `expires_at` column so the server knows when a ring should time out, even after restart:

```sql
ALTER TABLE calls ADD COLUMN IF NOT EXISTS expires_at TIMESTAMPTZ;
```

### Server — ring handler (`wsHandler.js`)

When a ring starts:

1. Insert into `calls` with `expires_at = NOW() + building_no_answer_timeout`.
2. For each device token being sent a push, insert a row into `call_delivery_attempts` with `delivery_state = 'queued'`.
3. After each push send succeeds, update the row to `push-sent` with `last_attempt_at = NOW()`.
4. If push fails (non-stale error), update to `push-failed` with `last_error`.
5. If push fails with a known-bad token, update to `push-failed`, delete the token (existing behavior), and note the error.

### Server — ack handler (`httpRoutes.js`)

When a delivery ack arrives at `/api/home/calls/:callId/ack`:

1. Continue writing to `call_delivery_acks` (existing behavior — this is the immutable event log).
2. Also update the corresponding `call_delivery_attempts` row's `delivery_state` and `acked_at`:
   ```sql
   UPDATE call_delivery_attempts
   SET delivery_state = $1, acked_at = NOW()
   WHERE call_id = $2 AND device_token = $3 AND attempt_number = (
       SELECT MAX(attempt_number) FROM call_delivery_attempts
       WHERE call_id = $2 AND device_token = $3
   )
   ```

### Server — startup recovery

On server start, query for calls that are still in `calling` status and not yet expired:

```sql
SELECT * FROM calls WHERE status = 'calling' AND expires_at > NOW()
```

For each such call, reconstruct the in-memory `pendingRing` and `activeCall` state from the DB. This makes ringing survive a restart.

If a call's `expires_at` has passed during the downtime, update it to `unanswered`.

## Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/sql/007-call-delivery-attempts.sql` | New migration |
| `vidrom-signaling-server/src/wsHandler.js` | Write delivery attempts on ring; update on push result |
| `vidrom-signaling-server/src/httpRoutes.js` | Update delivery attempts on ack |
| `vidrom-signaling-server/src/connectionState.js` | Add startup recovery from DB |
| `vidrom-signaling-server/src/server.js` | Call recovery on startup after DB connection |

## Verification

- [ ] Each push send creates a `call_delivery_attempts` row
- [ ] Push success → `push-sent`, push failure → `push-failed` with error
- [ ] Ack endpoint updates the latest attempt row's `delivery_state`
- [ ] Server restart during active ring → ring resumes from DB state
- [ ] Expired calls during downtime are marked `unanswered` on startup
- [ ] `calls.expires_at` is set correctly using building timeout
