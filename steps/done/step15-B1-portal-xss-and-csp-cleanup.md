# Step 15-B1 — Portal XSS And CSP Cleanup

## Problem

The portal pages still rely on inline scripts, inline event handlers, and broad HTML string rendering patterns. That makes a meaningful page-level CSP impractical and keeps avoidable XSS footguns in the code.

## What To Do

1. move portal page logic into external JavaScript files
2. remove inline event-handler attributes from static markup
3. replace action buttons rendered in HTML strings with `data-action` attributes plus delegated listeners
4. add a practical page-level CSP for the portal HTML

## Files Touched

| File | Change |
|------|--------|
| `vidrom-signaling-server/portals/admin.html` | Remove inline handlers and add page CSP |
| `vidrom-signaling-server/portals/management.html` | Remove inline handlers and add page CSP |
| `vidrom-signaling-server/portals/admin.js` | Externalize admin portal behavior |
| `vidrom-signaling-server/portals/management.js` | Externalize management portal behavior |

## Verification

- [ ] portal HTML no longer contains inline script blocks
- [ ] portal HTML no longer contains inline `onclick`/`onchange` handlers
- [ ] both pages load their behavior from external JavaScript files
- [ ] page-level CSP still allows Google Sign-In and portal API calls