# Notifications Flow & iOS Debug Guide

## Problem

Home **Android** receives push notifications from intercom → **works**.
Home **iOS** does NOT receive push notifications from intercom → **broken**.

---

## Current Notification Architecture

### Token Registration (Home App → Server)

| Token Type | Client Code | Server Endpoint | Server Storage |
|-----------|-------------|-----------------|----------------|
| **FCM** | `fcmService.js` → `getFCMToken()` | `POST /register-fcm-token` | `fcmTokens.set('home', token)` |
| **VoIP (PushKit)** | `voipService.js` → `registerVoipPush()` | `POST /register-voip-token` | `voipTokens.set('home', token)` |

Both tokens are registered on app startup (see `HomeScreen.js` lines ~120-155 for FCM, lines ~48-50 for VoIP).

### Call Flow: Intercom → Server → Home

```
Intercom App                    Signaling Server                 Home App
    │                                │                               │
    │── WS: { type: 'ring' } ─────→ │                               │
    │                                │                               │
    │                                ├── WS relay (if home online) ─→│
    │                                │                               │
    │                                ├── FCM push ─────────────────→ │  ← ONLY mechanism
    │                                │   (admin.messaging().send())  │
    │                                │                               │
    │                                │   ❌ VoIP push NEVER sent     │
    │                                │                               │
    │                                ├── setPendingRing() ───────────┤
    │                                │   (expires after 30s)         │
```

### Server Ring Handler (`wsHandler.js` lines 83-111)

When intercom sends `ring`, the server does:

1. `setPendingRing()` — marks ring as pending (survives WS disconnect, 30s TTL)
2. Relays `{ type: 'ring' }` via WebSocket to home (if connected)
3. Sends **FCM push** to home via `admin.messaging().send()` with:
   - `data: { type: 'incoming-call', callerName: 'Intercom' }`
   - `android: { priority: 'high' }`
   - `apns: { headers: { 'apns-priority': '10' }, payload: { aps: { contentAvailable: true, sound: 'default' } } }`

**VoIP push is NEVER sent** — the `voipTokens` map is populated but unused in the ring handler.

### Home App Reception

| Platform | Background/Killed | Foreground |
|----------|-------------------|------------|
| **Android** | `index.js` → `setBackgroundMessageHandler()` → shows Notifee full-screen notification | `HomeScreen.js` → `onForegroundMessage()` → connects WS |
| **iOS** | FCM `contentAvailable` → should wake app → `setBackgroundMessageHandler()` → `reportIncomingCall()` via CallKit | `onForegroundMessage()` → connects WS |

---

## Root Cause Analysis: Why iOS Fails

### Primary Issue: FCM `content-available` is unreliable for iOS incoming calls

The server sends FCM with `contentAvailable: true` which translates to APNs **silent push**. iOS treats these as low-priority background fetches:

- **Rate-limited** by iOS (especially on battery saver)
- **Not guaranteed to wake the app** when killed/suspended
- **Delayed or dropped** if the device is in Doze mode
- Apple explicitly states: "The system treats [content-available] as low-priority... and may throttle its delivery"

### Why Android Works

Android FCM with `priority: 'high'` is a **data-only high-priority message** — Android guarantees near-immediate delivery even when the app is killed, and the background message handler fires reliably.

### The Correct iOS Approach: VoIP Push (PushKit)

iOS has a **dedicated mechanism** for incoming calls: VoIP push via PushKit. The Home app already:
- ✅ Registers for VoIP push (`voipService.js`)
- ✅ Sends VoIP token to server (`POST /register-voip-token`)
- ✅ Has PushKit delegate in AppDelegate (`withVoipPushNotification.js`)
- ✅ Has `"voip"` in `UIBackgroundModes` (`app.json`)
- ✅ Has `reportIncomingCall()` ready to display CallKit UI

But the server **never sends a VoIP push** — it only uses `admin.messaging().send()` (Firebase/FCM).

Firebase Admin SDK **cannot send VoIP pushes**. VoIP push requires direct APNs communication using the PushKit topic (`voip` suffix on bundle ID).

---

## Token Storage: In-Memory Only (No Persistence)

Currently, both FCM and VoIP tokens are stored in `connectionState.js` as in-memory `Map` objects:

```js
const fcmTokens = new Map();   // Lost on server restart
const voipTokens = new Map();  // Lost on server restart
```

The `users` table has a `push_notification_token` column, but it's **unused** by the signaling server. Tokens are not persisted to the database.

---

## Debug Status Endpoint

The server exposes `GET /debug/status` (`httpRoutes.js` lines 94-107) which shows:
- WebSocket client connection state (home/intercom)
- FCM token presence (first 20 chars)
- VoIP token presence (first 20 chars)
- Pending ring state

```bash
curl http://signaling.vidrom.com:8080/debug/status
```

---

## Debug Checklist

### 1. Verify Token Registration

**Check the server has tokens:**
```bash
curl http://signaling.vidrom.com:8080/debug/status | python3 -m json.tool
```

Expected response should show both `fcmTokens.home` and `voipTokens.home` are non-null.

**If FCM token is null:**
- Check `HomeScreen.js` FCM setup — permission may have been denied
- Check Firebase project config — `GoogleService-Info.plist` must match bundle ID `com.vidrom.ai.home`
- Check APNs certificate is uploaded in Firebase Console → Cloud Messaging → iOS

**If VoIP token is null:**
- Check that `registerVoipPush()` is called on iOS
- Check VoIP push certificate in Apple Developer Portal
- Check device logs for `VoIP push token:` log line

### 2. Add Server-Side Logging

Add these logs to `wsHandler.js` ring handler to trace the notification flow:

```js
case 'ring': {
  setPendingRing();
  
  // Log all available tokens
  console.log(`[${id}] Ring received. FCM home token: ${fcmTokens.has('home') ? 'YES' : 'NO'}, VoIP home token: ${voipTokens.has('home') ? 'YES' : 'NO'}`);
  console.log(`[${id}] Home WS connected: ${clients.home ? clients.home.readyState === 1 : false}`);
  
  // ... existing relay code ...
  
  // Log FCM result
  admin.messaging().send({ ... })
    .then((messageId) => console.log(`[${id}] FCM push sent. Message ID: ${messageId}`))
    .catch((err) => console.error(`[${id}] FCM push FAILED: ${err.code} — ${err.message}`));
}
```

### 3. Add Notification Audit Table

Create a `push_notification_log` table to track every push attempt:

```sql
CREATE TABLE IF NOT EXISTS push_notification_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_role     VARCHAR(20)  NOT NULL,           -- 'home'
    target_platform VARCHAR(10),                      -- 'ios', 'android', or null if unknown
    push_type       VARCHAR(20)  NOT NULL,           -- 'fcm', 'voip', 'apns'
    token_prefix    VARCHAR(30),                      -- first 20 chars of token used
    payload         JSONB,                            -- notification payload sent
    status          VARCHAR(20)  NOT NULL DEFAULT 'sent',  -- 'sent', 'delivered', 'failed'
    error_message   TEXT,
    fcm_message_id  VARCHAR(255),
    trigger_type    VARCHAR(20)  NOT NULL,            -- 'ring', 'message', etc.
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_push_log_created ON push_notification_log (created_at DESC);
CREATE INDEX idx_push_log_status ON push_notification_log (status);
```

Then log every push attempt in the ring handler:

```js
// After FCM send attempt:
await query(`
  INSERT INTO push_notification_log 
    (target_role, push_type, token_prefix, status, error_message, fcm_message_id, trigger_type)
  VALUES ($1, $2, $3, $4, $5, $6, $7)
`, ['home', 'fcm', homeToken?.substring(0, 20), success ? 'sent' : 'failed', 
    errorMsg || null, messageId || null, 'ring']);
```

### 4. Add Client-Side Logging (iOS Home App)

**In `index.js`** — verify background handler fires:
```js
messaging().setBackgroundMessageHandler(async (remoteMessage) => {
  console.log('BG FCM received:', JSON.stringify(remoteMessage.data));
  // Also persist to AsyncStorage for later review:
  const log = await AsyncStorage.getItem('notif_debug_log') || '[]';
  const entries = JSON.parse(log);
  entries.push({ time: new Date().toISOString(), data: remoteMessage.data, source: 'bg_fcm' });
  await AsyncStorage.setItem('notif_debug_log', JSON.stringify(entries.slice(-20)));
  // ... existing code ...
});
```

**In `voipService.js`** — verify VoIP push fires:
```js
VoipPushNotification.addEventListener('notification', (notification) => {
  console.log('VoIP push received:', JSON.stringify(notification));
  // This will NOT fire until server actually sends VoIP push
});
```

**In `HomeScreen.js`** — add a debug button to view notification log:
```js
const showDebugLog = async () => {
  const log = await AsyncStorage.getItem('notif_debug_log') || '[]';
  Alert.alert('Notification Log', log);
};
```

### 5. Verify Firebase APNs Configuration

For FCM to deliver to iOS at all, Firebase needs a valid APNs key/certificate:

1. **Apple Developer Portal** → Keys → Create APNs key (or check existing)
2. **Firebase Console** → Project Settings → Cloud Messaging → iOS app
3. Upload the APNs Authentication Key (.p8 file)
4. Verify Team ID and Key ID are correct

Without this, `admin.messaging().send()` will fail silently or return an error for iOS tokens.

### 6. Check FCM Error Codes

Common FCM errors for iOS:
- `messaging/registration-token-not-registered` — token is stale/invalid
- `messaging/invalid-argument` — APNs config missing in Firebase
- `messaging/third-party-auth-error` — APNs key/cert not configured in Firebase Console
- `messaging/internal-error` — Firebase ↔ APNs connection issue

---

## Fix: Add VoIP Push Notification Support to Server

### Option A: Direct APNs VoIP Push (Recommended)

Use the `apn` npm package (or `@parse/node-apn`) to send VoIP push directly:

```bash
npm install @parse/node-apn
```

**Server setup (`server.js` or new `apnsService.js`):**
```js
const apn = require('@parse/node-apn');

const apnProvider = new apn.Provider({
  token: {
    key: '/path/to/APNs_AuthKey_XXXXXXXXXX.p8',
    keyId: 'YOUR_KEY_ID',
    teamId: 'YOUR_TEAM_ID',
  },
  production: false,  // true for production builds
});

async function sendVoipPush(voipToken, callerName) {
  const notification = new apn.Notification();
  notification.topic = 'com.vidrom.ai.home.voip';  // bundle ID + ".voip"
  notification.expiry = Math.floor(Date.now() / 1000) + 30;
  notification.priority = 10;
  notification.pushType = 'voip';
  notification.payload = { callerName };

  const result = await apnProvider.send(notification, voipToken);
  if (result.failed.length > 0) {
    console.error('VoIP push failed:', result.failed[0].response);
  }
  return result;
}
```

**Modified ring handler (`wsHandler.js`):**
```js
case 'ring': {
  setPendingRing();

  // ... existing WS relay ...

  // Send FCM for Android
  const homeToken = fcmTokens.get('home');
  if (homeToken) {
    admin.messaging().send({ /* existing FCM payload */ });
  }

  // Send VoIP push for iOS
  const homeVoipToken = voipTokens.get('home');
  if (homeVoipToken) {
    sendVoipPush(homeVoipToken, 'Intercom')
      .then(() => console.log(`[${id}] VoIP push sent to home`))
      .catch((err) => console.error(`[${id}] VoIP push failed:`, err));
  }
  break;
}
```

### Option B: Quick FCM Fix (Workaround, Less Reliable)

If you want to first verify FCM delivery works at all before implementing VoIP push, ensure the FCM payload includes a visible `notification` field (not just `data`):

```js
admin.messaging().send({
  token: homeToken,
  notification: {                    // ADD visible notification
    title: 'Incoming Call',
    body: 'Intercom is calling...',
  },
  data: {
    type: 'incoming-call',
    callerName: 'Intercom',
  },
  android: { priority: 'high' },
  apns: {
    headers: {
      'apns-priority': '10',
      'apns-push-type': 'alert',     // Explicitly set push type
    },
    payload: {
      aps: {
        alert: {                      // Visible alert instead of silent
          title: 'Incoming Call',
          body: 'Intercom is calling...',
        },
        sound: 'default',
        'content-available': 1,
      },
    },
  },
});
```

**⚠️ This is a workaround** — visible FCM notifications will show a banner but won't trigger CallKit's native incoming call screen. The proper fix is Option A (VoIP push).

---

## APNs Certificate Requirements

For VoIP push to work, you need:

1. **APNs Auth Key** (`.p8` file) — from Apple Developer Portal → Keys
   - Can be used for both regular and VoIP push
   - Note the **Key ID** and **Team ID**
2. **VoIP Services Certificate** — from Apple Developer Portal → Certificates → VoIP Services Certificate
   - Associated with `com.vidrom.ai.home`
3. **PushKit entitlement** — already present in `app.json`:
   ```json
   "UIBackgroundModes": ["voip", "remote-notification", "fetch"]
   ```

---

## Files Reference

| File | Role |
|------|------|
| `vidrom-signaling-server/src/wsHandler.js` | Ring handler, sends FCM push |
| `vidrom-signaling-server/src/httpRoutes.js` | Token registration endpoints, debug status |
| `vidrom-signaling-server/src/connectionState.js` | In-memory token storage |
| `vidrom-ai-home/voipService.js` | VoIP token registration + push listener |
| `vidrom-ai-home/fcmService.js` | FCM token registration + foreground listener |
| `vidrom-ai-home/callNotification.js` | Platform-specific notification display |
| `vidrom-ai-home/callkitService.js` | CallKit (iOS native call UI) |
| `vidrom-ai-home/index.js` | Background FCM handler + Notifee background events |
| `vidrom-ai-home/HomeScreen.js` | FCM setup, CallKit listeners, pending call check |
| `vidrom-ai-home/withVoipPushNotification.js` | Expo plugin — adds PushKit to AppDelegate |
| `vidrom-ai-home/app.json` | iOS entitlements (voip background mode) |
| `vidrom-signaling-server/sql/001-init.sql` | DB schema (notifications table exists but unused for push) |
