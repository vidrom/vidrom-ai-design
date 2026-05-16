# Step 12 — Cross-Repo Hardening And Maintainability

## Scope

Step 12 captures the highest-value improvements identified from a cross-repo review of:

- `vidrom-ai-home`
- `vidrom-ai-intercom`
- `vidrom-signaling-server`
- `vidrom-cdk`
- `vidrom-ai-design`

Unlike earlier steps that focused on a single feature area, this step is a coordinated hardening pass across application logic, infrastructure, security posture, documentation, and developer workflow.

This step should be treated as a planning umbrella. The items below can be implemented as sub-steps if needed.

---

## Goals

After Step 12:

1. The signaling path is correct and resilient under normal call load.
2. Secrets and infrastructure defaults are no longer embedded directly in source and deploy scripts.
3. Production traffic uses secure transport instead of plain HTTP and WS.
4. The design docs match the real message contract used by clients and server.
5. The repos have enough automated verification to catch regressions in call flow and infrastructure.
6. The mobile build workflow is reproducible for other developers and CI.

---

## Findings Summary

The review surfaced six main classes of improvement:

### 1. Correctness issues in the signaling server

- The main `ring` path in `vidrom-signaling-server/src/wsHandler.js` uses `ringTimeoutSec` before it is initialized.
- This is a hot-path bug in call setup and should be fixed first.

### 2. Secrets and credentials are embedded in source or deploy scripts

- JWT auth falls back to a hardcoded development secret.
- TURN credentials are embedded in app source.
- CDK and deploy scripts embed DB password and APNs identifiers.
- Local untracked secret files exist in the workspace, which is operationally risky even if they are not committed.

### 3. Production transport is not hardened

- Resolved by Step 12-A3: production defaults now target `https://` and `wss://` endpoints.
- Resolved by Step 12-A3: the home app no longer carries the production ATS insecure-load exception.
- Resolved by Step 12-A3: CDK outputs and ops scripts now advertise secure signaling endpoints.

### 4. Infrastructure is too permissive

- RDS is publicly accessible.
- CDK opens PostgreSQL access in a way that weakens the security boundary.
- TURN credentials are static and long-lived.

### 5. Documentation and code drift

- The design docs describe message names that do not match the implemented WebSocket contract.
- CDK documentation still reads like a default scaffold rather than project-specific guidance.

### 6. Testing coverage is too thin for the risk level of the system

- The signaling server has complex call-state logic without meaningful automated tests.
- The CDK repo still contains a placeholder example test.
- The mobile repos depend heavily on manual validation.

---

## Implementation Order

The items below are ordered by urgency and dependency.

| Priority | Area | Outcome |
|---|---|---|
| P0 | Signaling correctness | Fix the confirmed ring-path bug and add regression coverage |
| P0 | Secret handling | Remove hardcoded secrets and fail fast on missing required config |
| P0 | Transport security | Move apps and signaling to HTTPS and WSS |
| P1 | Infrastructure hardening | Reduce public exposure of DB and TURN components |
| P1 | Contract alignment | Make design docs and server message names consistent |
| P2 | Test coverage | Add server and infra tests for critical flows |
| P2 | Repo hygiene | Clean up README files, env setup, and reproducibility gaps |

---

## Sub-Step Layout

Step 12 is now split into concrete sub-step docs so the work can move in smaller, reviewable passes while this file remains the umbrella plan.

| Order | File | Focus | Notes |
|---|---|---|---|
| 1 | [step12-A1-signaling-correctness.md](step12-A1-signaling-correctness.md) | Fix the ring-path correctness bug | Highest priority, first landing item |
| 2 | [step12-A2-secret-management.md](step12-A2-secret-management.md) | Remove hardcoded secrets and validate config at startup | Foundation for later infra work |
| 3 | [step12-A3-https-wss-migration.md](step12-A3-https-wss-migration.md) | Move production traffic to HTTPS and WSS | Depends on stable config paths |
| 4 | [step12-B1-db-and-turn-hardening.md](step12-B1-db-and-turn-hardening.md) | Tighten DB exposure and TURN credentials | Best after A2 |
| 5 | [step12-B2-contract-alignment.md](step12-B2-contract-alignment.md) | Align docs with the live signaling contract | Can start immediately |
| 6 | [step12-C1-server-tests.md](step12-C1-server-tests.md) | Add critical server, infra, and targeted mobile tests | Starts with A1 regression coverage |
| 7 | [step12-C2-repo-docs-and-reproducibility.md](step12-C2-repo-docs-and-reproducibility.md) | Rewrite docs and verify build reproducibility | Best after config and infra changes settle |

## Execution Notes

The intent of the split is:

1. Keep Step 12 as the master plan and shared exit criteria.
2. Let implementation happen in smaller PRs with tighter scope.
3. Make dependencies visible so the riskiest work lands first.

---

## Exit Criteria

Step 12 is complete when all of the following are true:

- [ ] The confirmed ring-path bug is fixed and covered by a regression test
- [ ] No hardcoded production secret remains in committed app, server, or infra source
- [ ] Production app traffic uses HTTPS and WSS only
- [ ] DB and TURN security posture are improved to production-grade defaults
- [ ] Design docs reflect the real signaling message contract
- [ ] Critical signaling behavior is covered by meaningful automated tests
- [ ] Project READMEs and setup docs reflect the actual deployed architecture and workflow
