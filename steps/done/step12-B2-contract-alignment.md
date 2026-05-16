# Step 12-B2 — Contract Alignment (12.5)

## Problem

The implemented WebSocket message names differ from the documented contract.

Examples:

- docs say `ice-candidate`, code uses `candidate`
- docs say `end-call`, code uses `hangup`
- docs say `watch-start` / `watch-stop`, code uses `watch` / `watch-end`

This creates confusion during debugging and increases the chance of future client or server regressions.

## Dependencies

None — this can start immediately, but any code renames must ship as a coordinated cross-repo change.

## What To Do

1. Decide whether the code contract is canonical or whether names should be normalized.
2. Update `vidrom-ai-design` so the docs match the live implementation.
3. If renaming code is desired, do it as a compatibility-managed change across home app, intercom app, and signaling server together.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-ai-design/backend-architecture.md` | Align message names with the canonical contract |
| `vidrom-ai-design/spec.md` | Align message names with the canonical contract |
| any call-flow design docs that reference signaling messages | Remove drift from the documented API |

## Verification

- [x] Design docs and source use the same message names
- [x] Any renamed messages preserve backward compatibility during rollout if needed

## Outcome

- The live code contract was treated as canonical for this step.
- Design docs were aligned to `candidate`, `hangup`, `watch`, and `watch-end`.
- No runtime rename rollout was introduced in this step.