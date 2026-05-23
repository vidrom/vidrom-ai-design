# Step 15-A3 — Runtime Privilege And Secret Footprint

## Problem

The EC2 signaling service currently runs with more privilege than it needs and writes fetched secret material into the application directory.

That increases blast radius in two ways:

1. the Node signaling process has root-level execution when it only listens on an unprivileged port
2. Firebase and APNs secret files can linger on disk in the app directory longer than necessary

## What To Do

1. run the signaling service as a dedicated unprivileged system user
2. keep fetched secret files in a runtime directory under `/run`, not in the application tree
3. tighten systemd sandboxing for the signaling service
4. ensure secret files are cleaned up when the service stops or restarts

## Files Touched

| File | Change |
|------|--------|
| `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Create a dedicated runtime user and harden the systemd service |
| `vidrom-signaling-server/run.sh` | Write secret files to an ephemeral runtime directory and clean them up |

## Verification

- [ ] `vidrom-signaling` runs as the dedicated unprivileged user on EC2
- [ ] Firebase and APNs secret files are created under `/run/vidrom-signaling`
- [ ] the old app-directory secret file paths are no longer used in production
- [ ] the service still starts normally after a restart