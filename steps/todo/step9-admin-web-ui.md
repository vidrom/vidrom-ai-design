# Step 9 — Admin Web UI

## Goal

Build a web-based admin portal that allows authorized administrators to manage intercom devices through a browser instead of raw `curl` commands. The portal uses Google Sign-In for authentication and is restricted to a whitelist of admin Google accounts.

## Overview

- Single-page web app (HTML/CSS/JS) served by the signaling server on port 8080
- Google Sign-In (OAuth 2.0) for authentication
- Admin whitelist enforced on both client and server
- UI for: creating devices, viewing device list, generating provisioning codes, revoking devices
- No additional infrastructure — everything runs on the existing signaling server

## Authorized Admin Accounts

Only the following Google accounts may access the admin portal:

- `wwguyww@gmail.com`
- `ronenwes@gmail.com`

## Prerequisites

1. Step 7 completed (device auth, REST endpoints for create/provision/revoke)
2. A Google Cloud OAuth 2.0 Client ID (web application type) — created in the same Firebase/GCP project
3. Signaling server running on port 8080

---

## Step 9.1 — Create Google OAuth 2.0 Client ID

1. Go to [Google Cloud Console → APIs & Services → Credentials](https://console.cloud.google.com/apis/credentials)
2. Select the Vidrom project
3. Click **Create Credentials → OAuth client ID**
4. Application type: **Web application**
5. Name: `Vidrom Admin Portal`
6. Authorized JavaScript origins:
   - `http://localhost:8080` (for local dev)
   - `http://52.203.117.37:8080` (production server)
7. Click **Create** and note the **Client ID** (e.g., `123456789.apps.googleusercontent.com`)

---

## Step 9.2 — Add Admin Auth Middleware to Signaling Server

1. Add the admin whitelist and Google token verification to `server.js`:

   ```javascript
   // Admin whitelist
   const ADMIN_EMAILS = ['wwguyww@gmail.com', 'ronenwes@gmail.com'];

   // Verify Google ID token and check admin whitelist
   async function verifyAdminToken(req) {
     const authHeader = req.headers['authorization'];
     if (!authHeader || !authHeader.startsWith('Bearer ')) {
       return null;
     }
     const idToken = authHeader.slice(7);

     try {
       const decoded = await admin.auth().verifyIdToken(idToken);
       if (!ADMIN_EMAILS.includes(decoded.email)) {
         return null;
       }
       return decoded;
     } catch (err) {
       console.error('[AUTH] Token verification failed:', err.message);
       return null;
     }
   }
   ```

   Firebase Admin SDK (already installed) can verify Google ID tokens directly.

2. Wrap all `/api/admin/*` endpoints with the auth check:

   ```javascript
   // Before processing any /api/admin/* route:
   if (req.url.startsWith('/api/admin/')) {
     const adminUser = await verifyAdminToken(req);
     if (!adminUser) {
       res.writeHead(401, { 'Content-Type': 'application/json' });
       res.end(JSON.stringify({ error: 'Unauthorized' }));
       return;
     }
     console.log(`[ADMIN] Authenticated: ${adminUser.email}`);
   }
   ```

3. Add a new endpoint to list all devices:

   ```javascript
   // GET /api/admin/devices → returns array of all devices
   if (req.method === 'GET' && req.url === '/api/admin/devices') {
     const allDevices = listDevices(); // new function in devices.js
     res.writeHead(200, { 'Content-Type': 'application/json' });
     res.end(JSON.stringify(allDevices));
     return;
   }
   ```

---

## Step 9.3 — Add `listDevices()` to devices.js

```javascript
function listDevices() {
  return Array.from(devices.values());
}

module.exports = {
  createDevice,
  validateProvisioningCode,
  getDevice,
  revokeDevice,
  listDevices,  // new
};
```

---

## Step 9.4 — Serve Static Admin Page

1. Add a route in `server.js` to serve the admin HTML page:

   ```javascript
   const fs = require('fs');
   const path = require('path');

   // Serve admin portal
   if (req.method === 'GET' && req.url === '/admin') {
     const htmlPath = path.join(__dirname, 'admin.html');
     fs.readFile(htmlPath, 'utf8', (err, html) => {
       if (err) {
         res.writeHead(500);
         res.end('Error loading admin page');
         return;
       }
       res.writeHead(200, { 'Content-Type': 'text/html' });
       res.end(html);
     });
     return;
   }
   ```

2. The admin portal will be accessible at: `http://<SERVER_IP>:8080/admin`

---

## Step 9.5 — Create admin.html

Create `vidrom-signaling-server/admin.html` — a single-file SPA with embedded CSS and JS.

### Page Structure

```
┌─────────────────────────────────────────┐
│  Vidrom Admin Portal     [Sign In]      │
├─────────────────────────────────────────┤
│                                         │
│  ┌─ Create New Device ───────────────┐  │
│  │ Building ID: [________]           │  │
│  │ Device Name: [________]           │  │
│  │ [Create Device]                   │  │
│  │                                   │  │
│  │ Result: Code 482917               │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌─ Devices ─────────────────────────┐  │
│  │ Name       Building   Status  Act │  │
│  │ Front Door bldg-42   active  [Rev]│  │
│  │ Lobby      bldg-7    pending [Rev]│  │
│  │ Side Gate  bldg-42   revoked  --  │  │
│  └───────────────────────────────────┘  │
│                                         │
└─────────────────────────────────────────┘
```

### Key Implementation Details

1. **Google Sign-In:** Use Google Identity Services (GIS) library:
   ```html
   <script src="https://accounts.google.com/gsi/client" async></script>
   ```

   Initialize with the OAuth Client ID:
   ```javascript
   google.accounts.id.initialize({
     client_id: 'YOUR_CLIENT_ID.apps.googleusercontent.com',
     callback: handleCredentialResponse,
   });
   ```

2. **Token handling:**
   - On sign-in, Google returns a JWT `credential` (ID token)
   - Store it in memory for the session
   - Send it as `Authorization: Bearer <token>` on all API calls
   - Check email against the admin whitelist client-side (server also enforces)

3. **API calls from the admin UI:**

   ```javascript
   // Create device
   async function createDevice(buildingId, name) {
     const res = await fetch('/api/admin/devices', {
       method: 'POST',
       headers: {
         'Content-Type': 'application/json',
         'Authorization': `Bearer ${idToken}`,
       },
       body: JSON.stringify({ buildingId, name }),
     });
     return res.json();
   }

   // List devices
   async function listDevices() {
     const res = await fetch('/api/admin/devices', {
       headers: { 'Authorization': `Bearer ${idToken}` },
     });
     return res.json();
   }

   // Revoke device
   async function revokeDevice(deviceId) {
     const res = await fetch(`/api/admin/devices/${deviceId}/revoke`, {
       method: 'POST',
       headers: { 'Authorization': `Bearer ${idToken}` },
     });
     return res.json();
   }
   ```

4. **Device table:** Auto-refreshes after create/revoke operations. Shows:
   - Device name
   - Building ID
   - Status (pending / active / revoked) with color coding
   - Created date
   - Provisioning code (only shown for pending devices, if available)
   - Revoke button (disabled for already-revoked devices)

5. **Provisioning code display:** After creating a device, prominently display the 6-digit code so the admin can give it to the installer. Include a "Copy Code" button.

---

## Step 9.6 — Update CORS Headers

Update the CORS configuration in `server.js` to allow the `Authorization` header:

```javascript
res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
```

---

## Step 9.7 — Update CDK / Deployment

No infrastructure changes are needed — the admin page is served by the same HTTP server on port 8080. Just ensure `admin.html` is included in the deployment:

1. Update `deploy-server.sh` to copy `admin.html` alongside `server.js`:
   ```bash
   scp admin.html $SERVER_HOST:/opt/vidrom-signaling/
   ```

---

## Step 9.8 — Testing Checklist

1. **Google Sign-In:**
   - Navigate to `http://<SERVER_IP>:8080/admin`
   - Click Sign In → Google login popup appears
   - Sign in with `wwguyww@gmail.com` → access granted
   - Sign in with `ronenwes@gmail.com` → access granted
   - Sign in with any other account → access denied, error message shown

2. **Create Device:**
   - Fill in Building ID and Device Name → click Create
   - Verify provisioning code is displayed (6 digits)
   - Verify device appears in the device table with status "pending"

3. **List Devices:**
   - After creating multiple devices, verify all appear in the table
   - Verify status colors: pending=yellow, active=green, revoked=red

4. **Revoke Device:**
   - Click Revoke on an active device → confirm dialog
   - Verify status changes to "revoked"
   - Verify the button is now disabled

5. **Unauthenticated Access:**
   - Open admin page without signing in → only sign-in button visible
   - Try calling `/api/admin/devices` without auth header → 401 response

---

## Step 9.9 — Project Structure After Completion

```
vidrom-signaling-server/
├── server.js           # Updated: admin auth middleware, GET /api/admin/devices,
│                       #          serve /admin route, updated CORS
├── auth.js             # Unchanged
├── devices.js          # Updated: added listDevices()
├── admin.html          # New: single-file admin SPA with Google Sign-In
├── service-account.json
└── package.json        # Unchanged (no new dependencies)
```

---

## Security Notes

- **Double enforcement:** Admin whitelist is checked both client-side (UX) and server-side (security). The server is the authority.
- **Firebase Admin SDK** verifies Google ID tokens cryptographically — no shared secrets needed.
- **ID tokens expire** after ~1 hour. The GIS library handles silent refresh. If a token expires mid-session, API calls will return 401 and the UI should prompt re-login.
- **No new ports or services** — everything runs on the existing port 8080 HTTP server.
- **In-memory device store:** All devices are lost on server restart. This is acceptable for dev; production should use a persistent store (covered in a future step).

---

## Admin Flow Diagram

```
Admin (Browser)           Server (port 8080)         Google
     |                          |                      |
     |-- GET /admin ----------->|                      |
     |<-- admin.html -----------|                      |
     |                          |                      |
     |-- Google Sign-In --------|--------------------->|
     |<-- ID Token (JWT) -------|----------------------|
     |                          |                      |
     |-- GET /api/admin/devices |                      |
     |   Authorization: Bearer  |                      |
     |   <idToken> ------------>|                      |
     |                          |-- verifyIdToken() -->|
     |                          |<-- { email } --------|
     |                          |-- check whitelist    |
     |<-- [devices] ------------|                      |
     |                          |                      |
     |-- POST /api/admin/devices|                      |
     |   { buildingId, name } ->|                      |
     |<-- { deviceId, code } ---|                      |
```
