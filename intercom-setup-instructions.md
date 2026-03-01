# Vidrom Intercom — Device Setup Instructions

This guide explains how to create a new intercom system entry on the server and provision the physical intercom device with its authentication credentials.

---

## Prerequisites

- The Vidrom signaling server is running and accessible (default port `8080`)
- Access to the admin portal at `http://admin.vidrom.com:8080/admin` (requires an authorized Google account)
- The intercom Android app (vidrom-ai-intercom) is installed on the SBC device
- The SBC device has network connectivity to the signaling server

---

## Step 1 — Create a Device Entry via Admin Portal

1. Open the admin portal: **http://admin.vidrom.com:8080/admin**
2. Sign in with an authorized Google account (`wwguyww@gmail.com` or `ronenwes@gmail.com`)
3. In the **"Create New Device"** section:
   - Enter the **Building ID** (e.g., `building-42`)
   - Enter the **Device Name** (e.g., `Front Door`)
   - Click **"Create Device"**
4. A **6-digit provisioning code** will be displayed — copy it or write it down

**Important:** The provisioning code is one-time use. If it expires or is lost, create a new device entry.

---

## Step 2 — Provision the Intercom Device

1. **Power on the intercom SBC device** and ensure it is connected to the network.

2. **Launch the Vidrom Intercom app.** On first boot (or if no token is stored), the app will display the **"Vidrom Intercom Setup"** screen.

3. **Enter the 6-digit provisioning code** that was returned in Step 1.

4. **Tap "Activate Device".**

5. The app will contact the signaling server, exchange the code for a long-lived JWT authentication token, and store it locally on the device.

6. On success, the app automatically transitions to the main intercom UI and connects to the signaling server via WebSocket.

### What Happens Behind the Scenes

- The app sends `POST /api/devices/provision` with the 6-digit code.
- The server validates the code, marks the device as **active**, and returns a JWT (valid for 365 days).
- The app stores the JWT, deviceId, and buildingId in local AsyncStorage.
- On every subsequent boot, the app reads the saved token and uses it to authenticate with the signaling server — **no code re-entry needed**.

---

## Step 3 — Verify the Setup

After provisioning, confirm the device is working:

1. The intercom status should show **"Connected"** in the app header.
2. Press **"Ring"** on the intercom — the home app should receive the call notification.
3. Restart the intercom app — it should skip the setup screen and reconnect automatically.

---

## Revoking a Device

If a device is compromised, decommissioned, or needs to be re-provisioned:

1. Open the admin portal: **http://admin.vidrom.com:8080/admin**
2. Sign in with an authorized Google account
3. Find the device in the **Devices** table
4. Click the **"Revoke"** button and confirm

After revocation:
- The device's WebSocket connection will be refused on the next reconnect attempt.
- The intercom app will display an error and will not be able to operate until re-provisioned with a new code.

### Re-provisioning a Revoked Device

1. Create a new device entry (Step 1) to get a fresh provisioning code.
2. On the intercom device, clear the app data (or uninstall/reinstall) to remove the old stored token.
3. Follow Step 2 again with the new code.

---

## Troubleshooting

| Problem                              | Solution                                                                                     |
| ------------------------------------ | -------------------------------------------------------------------------------------------- |
| "Could not connect to server"        | Verify the server IP/port in `config.js` matches the running server. Check network connectivity. |
| "Invalid or expired code"            | The code was already used or doesn't exist. Create a new device entry (Step 1).               |
| Setup screen doesn't appear          | The device already has a stored token. Clear app data to reset, or the device is already provisioned. |
| "Device revoked or not found"        | The device was revoked by an admin. Re-provision with a new code (see Revoking section above). |
| "Token required" on WebSocket        | The app is missing its stored token. Clear app data and re-provision.                         |

---

## Security Notes

- **Provisioning codes are one-time use** — once exchanged for a JWT, the code is deleted.
- **JWTs expire after 365 days.** After expiry, the device must be re-provisioned.
- **In production:** Set the `JWT_SECRET` environment variable on the server to a strong, unique secret (do not use the default dev key). Add admin authentication to the REST endpoints. Replace the in-memory device store with a persistent database.

---

## Quick Reference

| Action             | Method | Endpoint                                   | Body                                |
| ------------------ | ------ | ------------------------------------------ | ----------------------------------- |
| Create device      | POST   | `/api/admin/devices`                       | `{ "buildingId": "...", "name": "..." }` |
| Provision device   | POST   | `/api/devices/provision`                   | `{ "code": "123456" }`             |
| Revoke device      | POST   | `/api/admin/devices/:deviceId/revoke`      | *(none)*                            |
