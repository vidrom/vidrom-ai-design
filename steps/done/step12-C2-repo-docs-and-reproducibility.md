# Step 12-C2 — Repo Docs And Build Reproducibility (12.7)

## Problem

Some project docs and repo artifacts still look like scaffolding or partial migration state.

Examples:

- CDK README is still generic boilerplate
- native folder generation is ignored in the mobile repos, which is fine only if the current scripts and config plugins fully reproduce all required native state

## Dependencies

Best done after A2, A3, and B1 so the docs describe the final config and deployment model instead of an intermediate state.

## What To Do

1. Rewrite the CDK README with actual project architecture, deploy commands, rollback notes, and secret prerequisites.
2. Document the production config contract for each repo.
3. Confirm that all required native behavior is reproducible from scripts, config plugins, patches, and tracked source.
4. If some native workaround still lives only in generated output, move it into tracked configuration.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-cdk/README.md` | Replace scaffold docs with the real deploy and rollback workflow |
| repo-specific setup docs or runbooks | Document actual config prerequisites and workflows |
| tracked mobile config/plugins | Capture the home iOS Podfile reproducibility fix in tracked source |

## Verification

- [x] A new developer can understand the deploy model from the checked-in docs
- [x] Native mobile builds are reproducible from tracked source and scripts
- [x] No critical build workaround depends on ignored generated folders

## Outcome

- Replaced the generic CDK README and empty signaling README with actual deploy, config, secret, and rollback documentation.
- Added dedicated README files for the home and intercom apps that document ports, run commands, env vars, and the generated-native workflow.
- Moved the home app's required iOS Podfile workarounds into a tracked Expo config plugin and added an automated test for the Podfile transform so the ignored native folders remain reproducible.