# Step 5 - Full-Screen Incoming Call UI

## Goal

Upgrade the basic push notifications from Step 4 to display a full-screen incoming call notification similar to WhatsApp, with Accept/Decline action buttons.

## Overview

- Use `@notifee/react-native` for advanced Android notification features
- Display full-screen notification when device is locked or app is in background
- Add Accept/Decline action buttons directly on the notification
- Handle notification actions to connect/reject calls

## Prerequisites

1. Step 4 completed (basic FCM push notifications working)
2. Firebase project configured with FCM

---

## Step 5.1 - Install Notifee Library

1. Install notifee for advanced notification handling:
   ```bash
   cd vidrom-ai-home
   npm install @notifee/react-native
   ```

2. Rebuild the development build:
   ```bash
   npx expo prebuild --clean
   npx expo run:android
   ```

---

## Step 5.2 - Configure Android Permissions

1. Add permissions to `android/app/src/main/AndroidManifest.xml`:
   ```xml
   <uses-permission android:name="android.permission.USE_FULL_SCREEN_INTENT" />
   <uses-permission android:name="android.permission.VIBRATE" />
   <uses-permission android:name="android.permission.WAKE_LOCK" />
   ```

2. These permissions allow:
   - `USE_FULL_SCREEN_INTENT` - Display notification over lock screen
   - `VIBRATE` - Vibration for incoming calls
   - `WAKE_LOCK` - Wake device when call arrives

---

## Step 5.3 - Create Notification Channel

1. Create a high-importance notification channel for incoming calls:
   ```javascript
   import notifee, { AndroidImportance } from '@notifee/react-native';
   
   async function createCallChannel() {
     await notifee.createChannel({
       id: 'incoming-call',
       name: 'Incoming Calls',
       importance: AndroidImportance.HIGH,
       sound: 'default',
       vibration: true,
     });
   }
   ```

2. Call `createCallChannel()` on app initialization (in App.js useEffect)

---

## Step 5.4 - Implement Full-Screen Notification Display

1. Create `callNotification.js` in vidrom-home:
   ```javascript
   import notifee, { AndroidImportance } from '@notifee/react-native';
   
   export async function showFullScreenCallNotification(callerName) {
     await notifee.displayNotification({
       title: 'Incoming Call',
       body: `Call from ${callerName}`,
       android: {
         channelId: 'incoming-call',
         importance: AndroidImportance.HIGH,
         fullScreenAction: {
           id: 'default',
         },
         pressAction: {
           id: 'default',
         },
         actions: [
           {
             title: 'Accept',
             pressAction: { id: 'accept' },
           },
           {
             title: 'Decline',
             pressAction: { id: 'decline' },
           },
         ],
         ongoing: true,
         autoCancel: false,
         category: AndroidCategory.CALL,
       },
     });
   }
   
   export async function cancelCallNotification() {
     await notifee.cancelAllNotifications();
   }
   ```

---

## Step 5.5 - Update Background Message Handler

1. Modify `index.js` to show full-screen notification:
   ```javascript
   import messaging from '@react-native-firebase/messaging';
   import { showFullScreenCallNotification } from './callNotification';
   
   messaging().setBackgroundMessageHandler(async (remoteMessage) => {
     if (remoteMessage.data.type === 'incoming-call') {
       await showFullScreenCallNotification(remoteMessage.data.callerName || 'Intercom');
     }
   });
   ```

2. Update foreground handler in App.js:
   ```javascript
   messaging().onMessage(async (remoteMessage) => {
     if (remoteMessage.data.type === 'incoming-call') {
       await showFullScreenCallNotification(remoteMessage.data.callerName || 'Intercom');
     }
   });
   ```

---

## Step 5.6 - Handle Notification Actions

1. Handle foreground notification events in App.js:
   ```javascript
   import notifee, { EventType } from '@notifee/react-native';
   
   useEffect(() => {
     const unsubscribe = notifee.onForegroundEvent(({ type, detail }) => {
       if (type === EventType.ACTION_PRESS) {
         if (detail.pressAction?.id === 'accept') {
           handleAccept();
           notifee.cancelAllNotifications();
         } else if (detail.pressAction?.id === 'decline') {
           handleDecline();
           notifee.cancelAllNotifications();
         }
       }
     });
     
     return () => unsubscribe();
   }, []);
   ```

2. Handle background events in `index.js`:
   ```javascript
   import notifee, { EventType } from '@notifee/react-native';
   
   notifee.onBackgroundEvent(async ({ type, detail }) => {
     if (type === EventType.ACTION_PRESS) {
       if (detail.pressAction?.id === 'accept') {
         // Store action in AsyncStorage for App.js to handle on launch
         await AsyncStorage.setItem('pendingCallAction', 'accept');
       } else if (detail.pressAction?.id === 'decline') {
         // Send decline via HTTP to server (WebSocket not available in background)
         await fetch(`http://${SERVER_IP}:${SERVER_PORT}/decline`, { method: 'POST' });
       }
       await notifee.cancelAllNotifications();
     }
   });
   ```

---

## Step 5.7 - Handle App Launch from Notification

1. Check for pending call action on app start:
   ```javascript
   useEffect(() => {
     const checkPendingAction = async () => {
       const action = await AsyncStorage.getItem('pendingCallAction');
       if (action === 'accept') {
         await AsyncStorage.removeItem('pendingCallAction');
         // Wait for WebSocket connection, then handle accept
         setCallState('in-call');
         // ... initiate WebRTC connection
       }
     };
     checkPendingAction();
   }, []);
   ```

2. Get initial notification that launched the app:
   ```javascript
   useEffect(() => {
     notifee.getInitialNotification().then((initialNotification) => {
       if (initialNotification?.pressAction?.id === 'accept') {
         // Handle accept from cold start
       }
     });
   }, []);
   ```

---

## Step 5.8 - Cancel Notification on Call State Change

1. Cancel the incoming call notification when:
   - User accepts the call (from app UI)
   - User declines the call (from app UI)
   - Call is answered/declined from another device
   - Intercom cancels the ring
   - Call times out

   ```javascript
   // In handleAccept/handleDecline functions
   import { cancelCallNotification } from './callNotification';
   
   const handleAccept = async () => {
     await cancelCallNotification();
     // ... existing accept logic
   };
   
   const handleDecline = () => {
     cancelCallNotification();
     // ... existing decline logic
   };
   ```

---

## Step 5.9 - Testing Checklist

1. **Locked Screen Test:**
   - Lock device, trigger ring from intercom
   - Verify full-screen notification appears over lock screen

2. **Background Test:**
   - Minimize app, trigger ring from intercom
   - Verify heads-up notification with Accept/Decline buttons

3. **Killed State Test:**
   - Force close app, trigger ring from intercom
   - Verify full-screen notification appears
   - Tap Accept → verify app opens and call connects

4. **Action Buttons Test:**
   - From notification, tap Accept → verify call starts
   - From notification, tap Decline → verify call is rejected

5. **Notification Dismissal:**
   - Accept from app UI → verify notification disappears
   - Intercom hangs up → verify notification disappears

---

## Step 5.10 - Project Structure After Completion

```
vidrom-ai/
├── vidrom-ai-home/
│   ├── App.js              # Updated with notifee event handlers
│   ├── callNotification.js # New: full-screen notification logic
│   ├── fcmService.js       # FCM token management (from Step 4)
│   ├── index.js            # Updated: background event handlers
│   └── android/
│       └── app/
│           └── src/main/
│               └── AndroidManifest.xml  # Updated with permissions
```

---

## Summary

| Feature | Implementation |
|---------|----------------|
| Full-screen notification | notifee with `fullScreenAction` |
| Action buttons | notifee `actions` array |
| Foreground events | `notifee.onForegroundEvent()` |
| Background events | `notifee.onBackgroundEvent()` |
| Cold start handling | `notifee.getInitialNotification()` |

---

## Next Step

See [step6.md](step6.md) for intercom device authentication.
