# Step: Configure APNs VoIP Push for iOS Notifications

## What was done (code)

The server now supports sending **VoIP push notifications** via APNs (Apple's recommended method for incoming calls). Changes:

1. **`@parse/node-apn`** installed on `vidrom-signaling-server`
2. **`src/apnsService.js`** created — initializes APNs provider, sends VoIP pushes
3. **`src/wsHandler.js`** ring handler — sends VoIP push to iOS alongside FCM for Android
4. **`src/server.js`** — calls `initAPNs()` on startup
5. **`src/httpRoutes.js`** — `/debug/status` now shows `apns.ready` field

## What you need to do (one-time setup)

### 1. Create an APNs Authentication Key

1. Go to [Apple Developer Portal → Keys](https://developer.apple.com/account/resources/authkeys/list)
2. Click **+** → Enable **Apple Push Notifications service (APNs)**
3. Download the `.p8` file (e.g., `AuthKey_ABC123.p8`)
4. Note the **Key ID** (shown on the key page, e.g., `ABC123`)
5. Note your **Team ID** from [Membership Details](https://developer.apple.com/account#MembershipDetailsCard)

### 2. Place the key on the server

Copy the `.p8` file to the signaling server root:

```bash
cp AuthKey_ABC123.p8 vidrom-signaling-server/apns-key.p8
```

> **Do NOT commit this file to git.** Add `apns-key.p8` to `.gitignore`.

### 3. Set environment variables

For **local development:**
```bash
export APN_KEY_ID="ABC123"          # Your Key ID from step 1
export APN_TEAM_ID="YOUR_TEAM_ID"  # Your Apple Team ID
export APN_PRODUCTION="false"       # Use APNs sandbox for dev builds
```

For **production (EC2):**
```bash
export APN_KEY_ID="ABC123"
export APN_TEAM_ID="YOUR_TEAM_ID"
export APN_PRODUCTION="true"        # Use APNs production for App Store / TestFlight builds
```

### 4. Verify it works

After starting the server, check:
```bash
curl http://signaling.vidrom.com:8080/debug/status | python3 -m json.tool
```

You should see `"apns": { "ready": true }`. If `false`, check the server logs for `[APNs]` messages.

### 5. Update CDK / deploy scripts

If deploying via CDK/SSM, add the env vars to the deploy script and copy the `.p8` file to the EC2 instance.

## Environment Variables Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `APN_KEY_ID` | (none — required) | APNs auth key ID from Apple Developer Portal |
| `APN_TEAM_ID` | (none — required) | Apple Developer Team ID |
| `APN_KEY_PATH` | `./apns-key.p8` | Path to the .p8 key file |
| `APN_BUNDLE_ID` | `com.vidrom.ai.home` | iOS app bundle identifier |
| `APN_PRODUCTION` | `false` | Set `true` for App Store / TestFlight builds |
