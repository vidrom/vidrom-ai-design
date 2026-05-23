# Step 15-A4 — Log Minimization

## Problem

Current server and portal API logs still print full operator emails, resident IDs, apartment IDs, call IDs, and other stable identifiers in several success and error paths.

That is not necessary for routine diagnostics and increases the amount of sensitive data retained in logs.

## What To Do

1. redact operator emails in portal API logs
2. redact resident, apartment, intercom, and call identifiers in high-volume signaling logs
3. log summarized error messages instead of dumping full error objects in common request paths
4. avoid logging secret-adjacent configuration details like APNs key IDs unless they are strictly necessary

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/src/logging.js` | Add redaction helpers for EC2 runtime logs |
| `vidrom-signaling-server/lambda/logging.js` | Add redaction helpers for portal Lambda logs |
| `vidrom-signaling-server/src/httpRoutes.js` | Redact high-volume resident/intercom identifiers |
| `vidrom-signaling-server/src/apnsService.js` | Remove avoidable APNs configuration details from logs |
| `vidrom-signaling-server/lambda/handler.js` | Redact operator emails in admin/management request logs |

## Verification

- [ ] portal request logs no longer print full operator emails
- [ ] high-volume resident call logs no longer print full stable identifiers
- [ ] common error logs prefer concise message text over full object dumps