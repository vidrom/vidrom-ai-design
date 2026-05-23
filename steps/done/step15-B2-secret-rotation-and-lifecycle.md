# Step 15-B2 — Secret Rotation And Lifecycle

## Problem

The stack already reads runtime secrets from Secrets Manager, but long-lived services still need a disciplined way to pick up rotated values without relying on ad-hoc manual restarts.

## What To Do

1. add an EC2-side refresh mechanism that periodically restarts the signaling service and coturn so they refetch secret values
2. keep the refresh mechanism separate from application code so it can be inspected and invoked manually
3. leave Lambda secret resolution compatible with cold-start refresh behavior

## Files Touched

| File | Change |
|------|--------|
| `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Add secret refresh helper and daily systemd timer |

## Verification

- [ ] EC2 instances install a dedicated secret-refresh helper script
- [ ] the timer is enabled so rotated Secrets Manager values are eventually picked up without manual SSH work
- [ ] signaling and coturn restart cleanly when the refresh helper runs