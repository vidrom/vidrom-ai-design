# Step 11-B4 — Intercom Visitor Feedback During Ringing (11.11)

## Problem

The intercom `InCallScreen` shows only "Ringing..." while the visitor waits. The visitor has no idea whether anyone was actually reached. With device acks flowing (Phase A) and retries (B3), the server now has real-time delivery progress it can push to the intercom.

## Dependencies

- **B3** (retry orchestration) — the retry orchestrator generates the `noResponse` signal
- **Phase A** (device acks) — ack events drive the `devicesConfirmed` count

## What To Do

### Server — ring progress events

Push `ring-progress` messages to the intercom's WebSocket as delivery data comes in:

1. **After initial push fanout** (in the ring handler):
   ```json
   { "type": "ring-progress", "callId": "...", "devicesNotified": 3 }
   ```
2. **When a device ack arrives** (in the ack HTTP handler or via a callback):
   ```json
   { "type": "ring-progress", "callId": "...", "devicesConfirmed": 1 }
   ```
   Send this on the first ack only (or update the count on each subsequent ack).
3. **If no device acks after the retry window** (from the retry orchestrator):
   ```json
   { "type": "ring-progress", "callId": "...", "noResponse": true }
   ```

To send these, the ack handler needs to look up the call's `intercom_id` and find the intercom's WS connection.

### Intercom — `InCallScreen.js`

Update the ringing UI based on `ring-progress` messages:

| State | Display |
|-------|---------|
| Default (ring sent) | "Ringing apartment 5..." |
| `devicesNotified: N` | "Ringing apartment 5... (N devices notified)" |
| `devicesConfirmed: N` | "Ringing apartment 5... (N devices confirmed)" |
| `noResponse: true` | "No response from apartment 5 — try again or use door code" |

The `noResponse` state should remain until the ring timeout expires and the normal no-answer flow triggers.

### Implementation notes

- The `ring-progress` messages are informational — they don't affect call state.
- The intercom should handle missing `ring-progress` gracefully (just keep showing "Ringing...").
- `devicesNotified` counts actual push sends (excluding skipped sleep-mode devices).
- `devicesConfirmed` counts distinct devices that sent at least one ack event.

## Files To Touch

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/wsHandler.js` | Send `ring-progress` after push fanout |
| `vidrom-signaling-server/src/httpRoutes.js` | On ack, send `ring-progress` with updated confirmed count to intercom |
| `vidrom-signaling-server/src/retryOrchestrator.js` | Send `ring-progress` with `noResponse` after final retry |
| `vidrom-ai-intercom/InCallScreen.js` | Display delivery progress to visitor |

## Verification

- [ ] Intercom receives `devicesNotified` shortly after ring starts
- [ ] Intercom receives `devicesConfirmed` when first device acks
- [ ] Intercom receives `noResponse` if no device acks after retry window
- [ ] InCallScreen displays updated text for each progress state
- [ ] Missing `ring-progress` messages don't break the default "Ringing..." UI
