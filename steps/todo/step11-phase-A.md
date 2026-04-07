# Step 11 Phase A — Immediate Reliability Wins

## Scope

Phase A delivers the foundational reliability layer described in step 11. It covers sub-steps **11.1, 11.2, 11.5, 11.13, 11.14, 11.15, and 11.16**.

After Phase A, every ring has a tracked `callId`, the server prunes dead tokens reactively, zombie intercom sockets are detected, ring timeout is configurable per building, residents can reserve a call via HTTP before full reconnect, and call history flows into the existing `calls` and `audit_logs` tables.

---

## Implementation Order

The items are ordered so each builds on the previous one. Items 1–3 are standalone quick wins. Item 4 is the foundation, and items 5–6 depend on it.

| # | File | Sub-step | Dependencies |
|---|------|----------|-------------|
| 1 | [step11-A1-ws-heartbeat.md](step11-A1-ws-heartbeat.md) | WebSocket Ping/Pong Heartbeat (11.15) | None |
| 2 | [step11-A2-stale-token-cleanup.md](step11-A2-stale-token-cleanup.md) | Reactive Stale Token Cleanup (11.14) | None |
| 3 | [step11-A3-building-timeout.md](step11-A3-building-timeout.md) | Building-Level No-Answer Timeout (11.13) | None |
| 4 | [step11-A4-call-id.md](step11-A4-call-id.md) | Call ID + `calls`/`audit_logs` Integration (11.1 + 11.16) | None |
| 5 | [step11-A5-device-acks.md](step11-A5-device-acks.md) | Device Acknowledgement Events (11.2) | A4 |
| 6 | [step11-A6-http-accept.md](step11-A6-http-accept.md) | Direct HTTP Accept (11.5) | A4 |

## Rollback Safety

All changes are additive:

- The `callId` field is added to messages but old home apps that don't send it back will still work — the server treats a missing `callId` on accept/decline/hangup as "use the current active call for this apartment" (existing behavior).
- The `call_delivery_acks` table is new and only written to; nothing reads it that would break if it's empty.
- The `calls` and `audit_logs` inserts are new writes that don't affect the existing ring flow.
- The HTTP accept endpoint is opt-in — old app versions that don't call it still accept via WebSocket as before.
- The ping/pong heartbeat is transparent to clients that don't handle it (the WebSocket protocol replies to pings automatically at the frame level).
- Stale token cleanup is a safe deletion — the next time the app opens, it re-registers its current token.
