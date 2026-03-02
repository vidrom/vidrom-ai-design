# Step 4 - Simple Push Notifications from Intercom to Home App

## Goal

Enable the intercom app to send basic push notifications to the home app, allowing the resident to receive incoming call alerts even when the app is in the background or closed.

## Overview

- Use Firebase Cloud Messaging (FCM) for cross-platform push notifications
- The signaling server stores FCM tokens and sends push notifications
- The home app displays a simple notification when a ring arrives
- FCM tokens are linked to the authenticated user from Step 3

## Prerequisites

1. Step 3 completed (Google login working, Firebase project configured)
2. Service account key for server-side FCM

---

## Step 4.1 - Generate Firebase Service Account Key

1. Go to Firebase Console → Project Settings → Service Accounts
2. Click "Generate new private key"
3. Save as `service-account.json` in `vidrom-signaling-server/`
4. Add `service-account.json` to `.gitignore`

---

## Step 4.2 - Install FCM Dependencies in vidrom-home

1. Install FCM messaging package:
   ```bash
   cd vidrom-ai-home
   npx expo install @react-native-firebase/messaging
   ```
   Note: `@react-native-firebase/app` was already installed in Step 3.

2. Update `app.json` to include FCM plugin:
   ```json
   {
     "expo": {
       "plugins": [
         "@react-native-firebase/app",
         "@react-native-firebase/auth",
         "@react-native-firebase/messaging",
         ["@react-native-google-signin/google-signin", { ... }]
       ]
     }
   }
   ```

3. Rebuild the development build:
   ```bash
   npx expo prebuild --clean
   npx expo run:android
   ```

---

## Step 4.3 - Configure FCM Token Management in vidrom-home

1. Create a new file `fcmService.js` in vidrom-home:
   - Request notification permissions on app start
   - Get FCM device token
   - Listen for token refresh events
   - Send the token to the signaling server

2. On app startup:
   - Call `messaging().getToken()` to get the FCM token
   - Send token to signaling server via WebSocket message:
     ```json
     { "type": "register-fcm-token", "token": "<FCM_TOKEN>" }
     ```

3. Handle token refresh:
   - Listen to `messaging().onTokenRefresh()` 
   - Send updated token to server

---

## Step 4.4 - Update Signaling Server for Push Notifications

1. Install Firebase Admin SDK:
   ```bash
   cd vidrom-signaling-server
   npm install firebase-admin
   ```

2. Initialize Firebase Admin with service account:
   ```javascript
   const admin = require('firebase-admin');
   const serviceAccount = require('./service-account.json');
   
   admin.initializeApp({
     credential: admin.credential.cert(serviceAccount)
   });
   ```

3. Store FCM tokens:
   - Add a `fcmTokens` map to store tokens by role
   - Handle `register-fcm-token` message type:
     ```javascript
     case 'register-fcm-token':
       fcmTokens.set(client.role, message.token);
       break;
     ```

4. Send push notification when intercom rings:
   - Modify the `ring` message handler
   - Before relaying to home WebSocket, also send FCM notification:
     ```javascript
     case 'ring':
       const homeToken = fcmTokens.get('home');
       if (homeToken) {
         await admin.messaging().send({
           token: homeToken,
           data: {
             type: 'incoming-call',
             callerName: 'Intercom',
           },
           android: {
             priority: 'high',
           },
           apns: {
             headers: { 'apns-priority': '10' },
             payload: {
               aps: {
                 contentAvailable: true,
                 sound: 'default',
               }
             }
           }
         });
       }
       // Also relay via WebSocket as before
       break;
     ```

---

## Step 4.5 - Handle Incoming Notifications in vidrom-home

1. Handle foreground messages (show alert or update UI):
   ```javascript
   messaging().onMessage(async (remoteMessage) => {
     if (remoteMessage.data.type === 'incoming-call') {
       console.log('Incoming call notification received');
       // The existing WebSocket ring handler will show the call UI
     }
   });
   ```

2. Handle background/killed state messages:
   ```javascript
   // In index.js (before AppRegistry)
   messaging().setBackgroundMessageHandler(async (remoteMessage) => {
     console.log('Background message:', remoteMessage.data);
     // Basic notification will be shown automatically by FCM
   });
   ```

---

## Step 4.6 - Testing Checklist

1. **Foreground Test:**
   - Open home app, trigger ring from intercom
   - Verify notification is received (check console log)

2. **Background Test:**
   - Minimize home app, trigger ring from intercom
   - Verify basic notification appears in notification tray

3. **Killed State Test:**
   - Force close home app, trigger ring from intercom
   - Verify notification appears

4. **Tap Notification:**
   - Tap notification → verify app opens

---

## Step 4.7 - Project Structure After Completion

```
vidrom-ai/
├── vidrom-signaling-server/
│   ├── server.js           # Updated with FCM push logic
│   ├── service-account.json # Firebase service account
│   └── package.json        # Added firebase-admin
├── vidrom-ai-home/
│   ├── App.js              # Updated with notification handling
│   ├── fcmService.js       # New: FCM token management
│   ├── index.js            # Updated: background handler
│   └── android/
│       └── app/
│           └── google-services.json
└── vidrom-ai-intercom/
    └── android/
        └── app/
            └── google-services.json
```

---

## Summary of Message Types (Extended)

| Type | Direction | Description |
|------|-----------|-------------|
| `register-fcm-token` | home → server | Register FCM token for push notifications |
| `ring` | intercom → server → home | Ring the home (now also triggers FCM push) |
| *(existing types unchanged)* | | |

---

## Next Step

See [step5.md](step5.md) for full-screen incoming call UI implementation.
