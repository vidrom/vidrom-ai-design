# Step 15 — Security Hardening Follow-Up

## Scope

Step 15 captures the next security improvements that remain after resident auth hardening, operator auth hardening, HTTPS/WSS migration, and DB/TURN infrastructure tightening.

The highest remaining gaps are no longer identity-binding issues. They are now mostly about browser/API boundary hardening, abuse resistance, runtime blast-radius reduction, and portal frontend safety.

## Goals

After Step 15:

1. portal APIs only allow the intended browser origins and return a stricter security-header set
2. sensitive routes are harder to abuse through brute-force or noisy-client traffic
3. the EC2 signaling runtime has a smaller privilege and secret-exposure footprint
4. portal frontend code is safer to lock down with a practical CSP
5. logs avoid unnecessary exposure of operator identifiers and other sensitive fields
6. secret rotation and secret material handling are more disciplined

## Current Status

Local implementation is complete.

Production rollout and live verification are complete.

Verified in production:

1. portal pages load with the Step 15 CSP-oriented external script structure intact
2. portal API responses include the hardened CORS and security headers
3. API Gateway and Lambda-side throttling are deployed for portal endpoints
4. `vidrom-signaling.service` runs under the hardened `vidrom` systemd profile
5. runtime secret material is recreated under `/run/vidrom-signaling`
6. `vidrom-secret-refresh.timer` is enabled and functioning
7. the EC2 deploy path was re-run successfully after a missing `run.sh` file was detected during refresh verification, and production health recovered cleanly

Step 15 is complete. The production closeout record is in [step15-final-production-cutover.md](../done/step15-final-production-cutover.md).

## Implementation Order

| Order | File | Focus | Notes |
|---|---|---|---|
| 1 | [step15-A1-portal-cors-and-security-headers.md](../done/step15-A1-portal-cors-and-security-headers.md) | Restrict portal API origins and add response security headers | Completed locally |
| 2 | [step15-A2-rate-limiting-and-abuse-controls.md](../done/step15-A2-rate-limiting-and-abuse-controls.md) | Add rate limiting on portal and provisioning-sensitive endpoints | Completed locally |
| 3 | [step15-A3-runtime-privilege-and-secret-footprint.md](../done/step15-A3-runtime-privilege-and-secret-footprint.md) | Reduce EC2 runtime privileges and tighten secret file handling | Completed locally |
| 4 | [step15-A4-log-minimization.md](../done/step15-A4-log-minimization.md) | Remove avoidable sensitive identifiers from logs | Completed locally |
| 5 | [step15-B1-portal-xss-and-csp-cleanup.md](../done/step15-B1-portal-xss-and-csp-cleanup.md) | Reduce `innerHTML` and inline handlers so a stronger CSP is practical | Completed locally |
| 6 | [step15-B2-secret-rotation-and-lifecycle.md](../done/step15-B2-secret-rotation-and-lifecycle.md) | Improve secret rotation and operational lifecycle controls | Completed locally |
| 7 | [step15-final-production-cutover.md](../done/step15-final-production-cutover.md) | Record the completed production rollout, verification, and recovery note | Completed |

## Notes

This step is intentionally ordered by leverage.

The first items reduce exposed surface area quickly. The later items are still worthwhile, but they either touch more code, require more coordination, or are better landed after the easier hardening wins are in place.