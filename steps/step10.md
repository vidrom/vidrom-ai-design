# Step 10 — iOS Support for Home App

## Goal

Add full iOS support to the vidrom-home app so residents using iPhones can receive incoming calls, view the intercom camera, and interact with the system identically to Android users. This includes Apple Sign-In as an alternative login method, APNs push notifications via FCM, iOS-native incoming call UI via CallKit, and building/deploying to a physical iPhone.

## Overview

- Add Apple Sign-In alongside existing Google Sign-In
- Configure APNs for FCM push notifications on iOS
- Replace Android-only Notifee full-screen notification with CallKit for iOS incoming calls
- Update `app.json` with iOS-specific configuration and entitlements
- Add Expo config plugin for iOS background modes
- Build and deploy to a physical iPhone via Xcode or EAS
- No changes needed to the signaling server — APNs payload is already configured (Step 4)

## Prerequisites

1. Steps 1–9 completed (Android home app fully functional)
2. macOS machine with Xcode 15+ installed (required for iOS builds)
3. Apple Developer account (paid, $99/year) enrolled at [developer.apple.com](https://developer.apple.com)
4. Physical iPhone for testing (push notifications do not work on Simulator)
5. Firebase project with iOS app registered

---

## Step 10.1 — Register iOS App in Firebase

1. Go to [Firebase Console](https://console.firebase.google.com/) → Project Settings → General
2. Click **Add app** → **iOS**
3. Enter bundle ID: `com.vidrom.ai.home`
4. App nickname: `Vidrom Home iOS`
5. Download `GoogleService-Info.plist`
6. Place the file at: `vidrom-ai-home/GoogleService-Info.plist`

7. Update `app.json` to reference the iOS Firebase config:
   ```json
   {
     "expo": {
       "ios": {
         "googleServicesFile": "./GoogleService-Info.plist"
       }
     }
   }
   ```

---

## Step 10.2 — Configure APNs for FCM on iOS

FCM on iOS requires Apple Push Notification service (APNs) credentials. There are two methods — **APNs Authentication Key** (recommended) or APNs certificate. Use the key method:

1. Go to [Apple Developer → Keys](https://developer.apple.com/account/resources/authkeys/list)
2. Click **+** to create a new key
3. Name: `Vidrom APNs Key`
4. Check **Apple Push Notifications service (APNs)**
5. Click **Continue** → **Register**
6. Download the `.p8` key file (e.g., `AuthKey_ABC123.p8`) — you can only download once
7. Note the **Key ID** (e.g., `ABC123`) and your **Team ID** (visible in Apple Developer membership page)

8. Upload the key to Firebase:
   - Firebase Console → Project Settings → Cloud Messaging tab
   - Under **Apple app configuration**, click **Upload** next to APNs Authentication Key
   - Upload the `.p8` file, enter Key ID and Team ID

9. This enables Firebase to send push notifications to iOS devices via APNs.

---

## Step 10.3 — Add Apple Sign-In

### 10.3.1 — Enable Apple Sign-In in Firebase

1. Firebase Console → Authentication → Sign-in method
2. Click **Add new provider** → **Apple**
3. Enable it — no extra configuration needed for native iOS apps (Firebase handles it automatically)

### 10.3.2 — Configure Apple Developer Portal

1. Go to [Certificates, Identifiers & Profiles](https://developer.apple.com/account/resources/)
2. Under **Identifiers**, find or create the App ID for `com.vidrom.ai.home`
3. Enable the **Sign in with Apple** capability
4. If using Sign in with Apple on the web (not needed here), configure the Services ID

### 10.3.3 — Install Apple Authentication Package

```bash
cd vidrom-ai-home
npx expo install expo-apple-authentication
```

### 10.3.4 — Update app.json

Add the Apple Sign-In plugin and entitlement:

```json
{
  "expo": {
    "ios": {
      "supportsTablet": true,
      "bundleIdentifier": "com.vidrom.ai.home",
      "infoPlist": {
        "NSMicrophoneUsageDescription": "Vidrom Home needs microphone access for audio calls",
        "UIBackgroundModes": ["voip", "remote-notification", "fetch"]
      },
      "entitlements": {
        "com.apple.developer.pushnotification": true
      },
      "googleServicesFile": "./GoogleService-Info.plist"
    },
    "plugins": [
      "expo-apple-authentication"
    ]
  }
}
```

### 10.3.5 — Update authService.js

Add Apple Sign-In support alongside Google:

```javascript
import { Platform } from 'react-native';
import auth from '@react-native-firebase/auth';
import { GoogleSignin } from '@react-native-google-signin/google-signin';
import * as AppleAuthentication from 'expo-apple-authentication';
import { OAuthProvider } from '@react-native-firebase/auth';

// Configure Google Sign-In (call once on app start)
export function configureGoogleSignIn() {
  GoogleSignin.configure({
    webClientId: 'YOUR_WEB_CLIENT_ID.apps.googleusercontent.com',
  });
}

// Sign in with Google
export async function signInWithGoogle() {
  await GoogleSignin.hasPlayServices({ showPlayServicesUpdateDialog: true });
  const response = await GoogleSignin.signIn();

  if (response.type === 'cancelled') {
    throw new Error('Sign-in was cancelled');
  }

  const { idToken } = response.data;
  if (!idToken) {
    throw new Error('No ID token returned from Google Sign-In');
  }

  const googleCredential = auth.GoogleAuthProvider.credential(idToken);
  return auth().signInWithCredential(googleCredential);
}

// Sign in with Apple (iOS only)
export async function signInWithApple() {
  // Request credentials from Apple
  const appleCredential = await AppleAuthentication.signInAsync({
    requestedScopes: [
      AppleAuthentication.AppleAuthenticationScope.FULL_NAME,
      AppleAuthentication.AppleAuthenticationScope.EMAIL,
    ],
  });

  // Create a Firebase credential from the Apple response
  const { identityToken, authorizationCode } = appleCredential;
  if (!identityToken) {
    throw new Error('No identity token returned from Apple Sign-In');
  }

  const provider = new OAuthProvider('apple.com');
  const credential = provider.credential({
    idToken: identityToken,
    rawNonce: authorizationCode,
  });

  return auth().signInWithCredential(credential);
}

// Check if Apple Sign-In is available (iOS 13+)
export async function isAppleSignInAvailable() {
  if (Platform.OS !== 'ios') return false;
  return await AppleAuthentication.isAvailableAsync();
}

// Sign out
export async function signOut() {
  if (Platform.OS === 'android') {
    await GoogleSignin.revokeAccess();
  }
  await auth().signOut();
}

// Get current user
export function getCurrentUser() {
  return auth().currentUser;
}

// Listen to auth state changes
export function onAuthStateChanged(callback) {
  return auth().onAuthStateChanged(callback);
}
```

### 10.3.6 — Update LoginScreen.js

Add Apple Sign-In button for iOS users:

```javascript
import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ActivityIndicator,
  Alert,
  Platform,
} from 'react-native';
import { signInWithGoogle, signInWithApple, isAppleSignInAvailable } from './authService';

export default function LoginScreen() {
  const [loading, setLoading] = useState(false);
  const [appleAvailable, setAppleAvailable] = useState(false);

  useEffect(() => {
    isAppleSignInAvailable().then(setAppleAvailable);
  }, []);

  const handleGoogleSignIn = async () => {
    setLoading(true);
    try {
      await signInWithGoogle();
    } catch (error) {
      console.error('Google Sign-In error:', error);
      Alert.alert('Sign-In Error', error.message);
    } finally {
      setLoading(false);
    }
  };

  const handleAppleSignIn = async () => {
    setLoading(true);
    try {
      await signInWithApple();
    } catch (error) {
      if (error.code === 'ERR_REQUEST_CANCELED') {
        // User cancelled — not an error
      } else {
        console.error('Apple Sign-In error:', error);
        Alert.alert('Sign-In Error', error.message);
      }
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Vidrom Home</Text>
      <Text style={styles.subtitle}>Sign in to receive calls</Text>

      <TouchableOpacity
        style={styles.googleButton}
        onPress={handleGoogleSignIn}
        disabled={loading}
      >
        {loading ? (
          <ActivityIndicator color="#fff" />
        ) : (
          <Text style={styles.buttonText}>Sign in with Google</Text>
        )}
      </TouchableOpacity>

      {appleAvailable && (
        <TouchableOpacity
          style={styles.appleButton}
          onPress={handleAppleSignIn}
          disabled={loading}
        >
          <Text style={styles.appleButtonText}> Sign in with Apple</Text>
        </TouchableOpacity>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  // ... existing styles ...
  appleButton: {
    backgroundColor: '#000',
    paddingVertical: 15,
    paddingHorizontal: 30,
    borderRadius: 8,
    minWidth: 250,
    alignItems: 'center',
    marginTop: 12,
  },
  appleButtonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

---

## Step 10.4 — iOS Incoming Call UI with CallKit

### Rationale

Android uses Notifee for full-screen lock-screen notifications. iOS does not have an equivalent API. Instead, iOS uses **CallKit** — the native VoIP calling framework — to display the system incoming call screen (identical to a normal phone call). This is also required by Apple's App Store guidelines for VoIP apps.

### 10.4.1 — Install react-native-callkeep

```bash
cd vidrom-ai-home
npm install react-native-callkeep
```

### 10.4.2 — Update app.json for VoIP

Ensure the iOS background modes include `voip`:

```json
{
  "expo": {
    "ios": {
      "infoPlist": {
        "UIBackgroundModes": ["voip", "remote-notification", "fetch"]
      }
    }
  }
}
```

### 10.4.3 — Create callkitService.js

```javascript
import { Platform } from 'react-native';
import RNCallKeep from 'react-native-callkeep';
import { v4 as uuidv4 } from 'uuid';

let currentCallUUID = null;

// Initialize CallKit (call once on app start, iOS only)
export async function setupCallKit() {
  if (Platform.OS !== 'ios') return;

  const options = {
    ios: {
      appName: 'Vidrom Home',
      supportsVideo: true,
      maximumCallGroups: 1,
      maximumCallsPerCallGroup: 1,
    },
  };

  try {
    await RNCallKeep.setup(options);
    RNCallKeep.setAvailable(true);
  } catch (err) {
    console.error('CallKit setup error:', err);
  }
}

// Display the native incoming call UI
export function reportIncomingCall(callerName = 'Intercom') {
  if (Platform.OS !== 'ios') return null;

  currentCallUUID = uuidv4();
  RNCallKeep.displayIncomingCall(
    currentCallUUID,
    'Intercom',          // handle (phone number / identifier)
    callerName,          // caller name displayed
    'generic',           // handle type
    true                 // has video
  );
  return currentCallUUID;
}

// End the CallKit call (dismiss the native UI)
export function endCallKitCall() {
  if (Platform.OS !== 'ios' || !currentCallUUID) return;
  RNCallKeep.endCall(currentCallUUID);
  currentCallUUID = null;
}

// Register CallKit event listeners
export function registerCallKitListeners({ onAnswer, onEnd }) {
  if (Platform.OS !== 'ios') return () => {};

  RNCallKeep.addEventListener('answerCall', ({ callUUID }) => {
    console.log('CallKit: user answered call', callUUID);
    onAnswer(callUUID);
  });

  RNCallKeep.addEventListener('endCall', ({ callUUID }) => {
    console.log('CallKit: user ended call', callUUID);
    onEnd(callUUID);
  });

  return () => {
    RNCallKeep.removeEventListener('answerCall');
    RNCallKeep.removeEventListener('endCall');
  };
}

export function getCurrentCallUUID() {
  return currentCallUUID;
}
```

### 10.4.4 — Create VoIP Push Notification Handler (iOS)

iOS VoIP push notifications use a separate channel (PushKit) that can wake the app even when it's killed. This is required for CallKit to show the native call screen on the lock screen.

Install the VoIP push handler:

```bash
npm install react-native-voip-push-notification
```

Create `voipService.js`:

```javascript
import { Platform } from 'react-native';
import VoipPushNotification from 'react-native-voip-push-notification';
import { reportIncomingCall } from './callkitService';
import { SERVER_HTTP } from './config';

// Register for VoIP push notifications (iOS only)
export function registerVoipPush() {
  if (Platform.OS !== 'ios') return;

  VoipPushNotification.addEventListener('register', (voipToken) => {
    console.log('VoIP push token:', voipToken);
    // Send the VoIP token to the server alongside the FCM token
    registerVoipTokenWithServer(voipToken);
  });

  VoipPushNotification.addEventListener('notification', (notification) => {
    console.log('VoIP push notification received:', notification);
    // Display the native CallKit incoming call UI
    reportIncomingCall(notification.callerName || 'Intercom');
  });

  VoipPushNotification.addEventListener('didLoadWithEvents', (events) => {
    // Process any events that arrived before JS was ready
    if (!events || !Array.isArray(events) || events.length < 1) return;
    for (const event of events) {
      if (event.name === 'RNVoipPushRemoteNotificationsRegisteredEvent') {
        registerVoipTokenWithServer(event.data);
      } else if (event.name === 'RNVoipPushRemoteNotificationReceivedEvent') {
        reportIncomingCall(event.data?.callerName || 'Intercom');
      }
    }
  });

  VoipPushNotification.registerVoipToken();
}

async function registerVoipTokenWithServer(token) {
  try {
    await fetch(`${SERVER_HTTP}/register-voip-token`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ role: 'home', token }),
    });
    console.log('VoIP token sent to server');
  } catch (err) {
    console.error('Failed to register VoIP token:', err);
  }
}
```

> **Note:** VoIP pushes are the Apple-approved way to wake a killed app and trigger CallKit. Standard FCM data-only pushes on iOS do NOT reliably wake the app. For production, the server should send VoIP pushes directly via APNs for incoming calls. As an intermediate step, we can still use FCM for foreground/background calls and add VoIP push support for killed-state calls later.

---

## Step 10.5 — Update callNotification.js for Cross-Platform

The current `callNotification.js` is Android-only. Update it to support both platforms:

```javascript
import { Platform } from 'react-native';
import notifee, { AndroidImportance, AndroidCategory, AndroidVisibility, AndroidColor } from '@notifee/react-native';
import { reportIncomingCall, endCallKitCall } from './callkitService';

// Create a high-importance notification channel for incoming calls (Android only)
export async function createCallChannel() {
  if (Platform.OS !== 'android') return;

  await notifee.createChannel({
    id: 'incoming-call',
    name: 'Incoming Calls',
    importance: AndroidImportance.HIGH,
    sound: 'default',
    vibration: true,
    vibrationPattern: [1, 500, 200, 500, 200, 500, 200, 500],
    bypassDnd: true,
    lights: true,
    lightColor: AndroidColor.GREEN,
    visibility: AndroidVisibility.PUBLIC,
  });
}

// Display a full-screen incoming call notification
export async function showFullScreenCallNotification(callerName) {
  if (Platform.OS === 'ios') {
    // Use CallKit for the native iOS incoming call screen
    reportIncomingCall(callerName);
    return;
  }

  // Android: use Notifee full-screen notification
  await createCallChannel();

  await notifee.displayNotification({
    title: '📹 Incoming Video Call',
    body: `${callerName} is calling...`,
    android: {
      channelId: 'incoming-call',
      importance: AndroidImportance.HIGH,
      fullScreenAction: {
        id: 'default',
        launchActivity: 'default',
      },
      pressAction: {
        id: 'default',
      },
      actions: [
        {
          title: '✓ Accept',
          pressAction: { id: 'accept' },
        },
        {
          title: '✕ Decline',
          pressAction: { id: 'decline' },
        },
      ],
      ongoing: true,
      autoCancel: false,
      category: AndroidCategory.CALL,
      visibility: AndroidVisibility.PUBLIC,
      colorized: true,
      color: '#0d1b2a',
      timestamp: Date.now(),
      showTimestamp: true,
    },
  });
}

// Cancel all call notifications
export async function cancelCallNotification() {
  if (Platform.OS === 'ios') {
    endCallKitCall();
    return;
  }
  await notifee.cancelAllNotifications();
}
```

---

## Step 10.6 — Update App.js and HomeScreen.js for iOS

### 10.6.1 — Update App.js

Add CallKit setup on iOS:

```javascript
import React, { useEffect, useState } from 'react';
import { Platform, View, ActivityIndicator } from 'react-native';
import { configureGoogleSignIn, onAuthStateChanged } from './authService';
import { createCallChannel } from './callNotification';
import { setupCallKit } from './callkitService';
import LoginScreen from './LoginScreen';
import HomeScreen from './HomeScreen';
import styles from './styles';

export default function App() {
  const [user, setUser] = useState(null);
  const [initializing, setInitializing] = useState(true);

  useEffect(() => {
    configureGoogleSignIn();
    createCallChannel();          // No-op on iOS
    if (Platform.OS === 'ios') {
      setupCallKit();
    }
    const unsubscribe = onAuthStateChanged((u) => {
      setUser(u);
      if (initializing) setInitializing(false);
    });
    return unsubscribe;
  }, []);

  if (initializing) {
    return (
      <View style={styles.loadingContainer}>
        <ActivityIndicator size="large" color="#4285F4" />
      </View>
    );
  }

  if (!user) {
    return <LoginScreen />;
  }

  return <HomeScreen user={user} />;
}
```

### 10.6.2 — Update HomeScreen.js

Add CallKit event listeners for iOS accept/decline:

```javascript
import { Platform } from 'react-native';
import { registerCallKitListeners } from './callkitService';
import { registerVoipPush } from './voipService';

// Inside HomeScreen component, add this useEffect:
useEffect(() => {
  if (Platform.OS === 'ios') {
    registerVoipPush();
  }
}, []);

// CallKit listeners (iOS native answer/end)
useEffect(() => {
  if (Platform.OS !== 'ios') return;

  const unsubscribe = registerCallKitListeners({
    onAnswer: (callUUID) => {
      // User tapped Accept on the native iOS call screen
      connectForIncomingCall().then(() => {
        handleAccept();
      });
    },
    onEnd: (callUUID) => {
      // User tapped Decline/End on the native iOS call screen
      handleDecline();
    },
  });

  return unsubscribe;
}, [connectForIncomingCall, handleAccept, handleDecline]);
```

---

## Step 10.7 — Update app.json (Complete iOS Configuration)

The full updated `app.json`:

```json
{
  "expo": {
    "name": "Vidrom Home",
    "slug": "vidrom-ai-home",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "userInterfaceStyle": "light",
    "newArchEnabled": true,
    "splash": {
      "image": "./assets/splash-icon.png",
      "resizeMode": "contain",
      "backgroundColor": "#ffffff"
    },
    "ios": {
      "supportsTablet": true,
      "bundleIdentifier": "com.vidrom.ai.home",
      "googleServicesFile": "./GoogleService-Info.plist",
      "infoPlist": {
        "NSMicrophoneUsageDescription": "Vidrom Home needs microphone access for audio calls",
        "NSCameraUsageDescription": "Vidrom Home may access the camera for video calls",
        "UIBackgroundModes": ["voip", "remote-notification", "fetch"]
      },
      "entitlements": {
        "com.apple.developer.pushnotification": true
      }
    },
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/adaptive-icon.png",
        "backgroundColor": "#ffffff"
      },
      "edgeToEdgeEnabled": true,
      "package": "com.vidrom.ai.home",
      "googleServicesFile": "./google-services.json",
      "permissions": [
        "RECORD_AUDIO",
        "MODIFY_AUDIO_SETTINGS",
        "USE_FULL_SCREEN_INTENT",
        "VIBRATE",
        "WAKE_LOCK"
      ]
    },
    "web": {
      "favicon": "./assets/favicon.png"
    },
    "plugins": [
      [
        "expo-build-properties",
        {
          "android": {
            "minSdkVersion": 24
          },
          "ios": {
            "deploymentTarget": "15.0"
          }
        }
      ],
      "@react-native-firebase/app",
      "@react-native-firebase/auth",
      "@react-native-google-signin/google-signin",
      "@react-native-firebase/messaging",
      "expo-apple-authentication",
      "./withWakeScreen",
      "./withNotifeeRepo"
    ]
  }
}
```

---

## Step 10.8 — Update Signaling Server for VoIP Push (Optional Enhancement)

For reliable call delivery when the iOS app is killed, the server should send VoIP pushes directly via APNs. This is optional for initial iOS support (FCM handles foreground/background cases), but required for production.

### 10.8.1 — Add VoIP Token Storage

Add a new HTTP endpoint in `httpRoutes.js`:

```javascript
// POST /register-voip-token  { role, token }
else if (req.method === 'POST' && req.url === '/register-voip-token') {
  readBody(req).then(({ role, token }) => {
    if (role && token) {
      voipTokens.set(role, token);
      console.log(`[HTTP] VoIP token registered for "${role}"`);
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ ok: true }));
    } else {
      res.writeHead(400, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ error: 'role and token required' }));
    }
  });
}
```

### 10.8.2 — Send VoIP Push on Ring

In `wsHandler.js`, alongside the existing FCM push in the `ring` handler, add a VoIP push for iOS:

```javascript
// Send VoIP push for iOS (uses APNs directly via firebase-admin)
const voipToken = voipTokens.get('home');
if (voipToken) {
  // VoIP pushes are sent via APNs PushKit topic
  // The server needs the APN VoIP certificate or can use a library like 'apn'
  // For now, we rely on FCM which handles APNs for regular pushes
  console.log(`[${id}] VoIP push would be sent to home (token registered)`);
}
```

> **Production note:** For full killed-state support on iOS, you'll need to use a dedicated APNs library (e.g., `@parse/node-apn`) to send VoIP pushes with the `voip` topic suffix. This is because FCM does not support sending VoIP push notifications — only regular APNs. This can be addressed in a future step.

---

## Step 10.9 — Build and Run on iOS

### 10.9.1 — Prebuild for iOS

```bash
cd vidrom-ai-home
npx expo prebuild --platform ios --clean
```

This generates the `ios/` folder with the Xcode project, applying all config plugins.

### 10.9.2 — Install CocoaPods Dependencies

```bash
cd ios
pod install
cd ..
```

### 10.9.3 — Configure Signing in Xcode

1. Open `ios/VidromHome.xcworkspace` in Xcode
2. Select the **VidromHome** target
3. Go to **Signing & Capabilities** tab
4. Select your Apple Developer team
5. Ensure the bundle identifier is `com.vidrom.ai.home`
6. Verify these capabilities are present (added automatically by expo prebuild):
   - **Push Notifications**
   - **Background Modes** — Voice over IP, Remote notifications, Background fetch
   - **Sign in with Apple**

### 10.9.4 — Run on Device

```bash
npx expo run:ios --device
```

Or open the Xcode workspace and run from Xcode (select your connected iPhone as the target).

### 10.9.5 — Create run-ios.sh

Create a convenience script similar to `run-android.sh`:

```bash
#!/bin/bash
npx expo run:ios --device
```

---

## Step 10.10 — iOS Build for Distribution

### Using EAS Build

```bash
cd vidrom-ai-home
eas build --platform ios --profile development
```

### Using Firebase App Distribution

1. Build the IPA via Xcode (Product → Archive → Distribute → Ad Hoc)
2. Upload via Firebase CLI:
   ```bash
   firebase appdistribution:distribute ./VidromHome.ipa \
     --app="YOUR_IOS_FIREBASE_APP_ID" \
     --release-notes="Vidrom Home iOS v1" \
     --testers="wwguyww@gmail.com,ronenwes@gmail.com"
   ```

---

## Step 10.11 — Testing Checklist

### Authentication

1. **Google Sign-In on iOS:**
   - Tap "Sign in with Google" → Google login sheet appears
   - Sign in → verify user is authenticated and HomeScreen shows

2. **Apple Sign-In on iOS:**
   - Tap "Sign in with Apple" → native Apple sign-in prompt appears
   - Sign in with Face ID/Touch ID → verify user is authenticated
   - Verify the button only appears on iOS (hidden on Android)

3. **Sign Out:**
   - Tap Sign Out → verify return to login screen

### Push Notifications

4. **FCM Token Registration:**
   - Launch app on iPhone → verify FCM token is registered with server (check server logs)

5. **Foreground Push:**
   - App open → intercom rings → verify call notification handled and incoming call UI shown

6. **Background Push:**
   - App in background → intercom rings → verify iOS notification appears
   - Tap notification → verify app opens to incoming call screen

### CallKit (iOS)

7. **Incoming Call on Lock Screen:**
   - Device locked → intercom rings → verify native iOS call screen appears
   - Tap Accept → verify app opens and call connects
   - Tap Decline → verify call is rejected

8. **Incoming Call While Using Phone:**
   - Using another app → intercom rings → verify banner notification / CallKit UI appears

### WebRTC Calls

9. **Accept Call:**
   - Receive call on iPhone → Accept → verify two-way audio and one-way video (intercom camera → iPhone)

10. **Hang Up:**
    - During active call → tap Hang Up → verify call ends cleanly on both sides

11. **Watch Mode:**
    - From idle screen → tap Watch → verify intercom camera stream appears
    - Tap Stop Watching → verify stream stops and returns to idle

### Cross-Platform

12. **Android Unchanged:**
    - Verify all existing Android functionality still works after code changes
    - Full-screen Notifee notification still works on Android
    - Google Sign-In still works on Android

---

## Step 10.12 — Project Structure After Completion

```
vidrom-ai-home/
├── App.js                   # Updated: CallKit setup on iOS
├── app.json                 # Updated: iOS config, googleServicesFile, entitlements,
│                            #          background modes, expo-apple-authentication plugin
├── authService.js           # Updated: Apple Sign-In support, platform-aware signOut
├── callNotification.js      # Updated: Platform switch — CallKit on iOS, Notifee on Android
├── callkitService.js        # New: CallKit wrapper for iOS incoming call UI
├── voipService.js           # New: VoIP push notification registration (iOS)
├── LoginScreen.js           # Updated: Apple Sign-In button on iOS
├── HomeScreen.js            # Updated: CallKit listeners, VoIP push registration
├── fcmService.js            # Unchanged (already handles iOS permission)
├── useCallManager.js        # Unchanged (platform-agnostic WebRTC)
├── IncomingCallScreen.js    # Unchanged (used for in-app UI on both platforms)
├── config.js                # Unchanged
├── styles.js                # Updated: Apple button styles
├── index.js                 # Unchanged (Notifee background events are Android-only)
├── GoogleService-Info.plist # New: Firebase config for iOS
├── google-services.json     # Unchanged (Android Firebase config)
├── package.json             # Updated: new dependencies
├── run-ios.sh               # New: convenience build script for iOS
├── build.sh                 # Unchanged
├── android/                 # Unchanged
└── ios/                     # New: generated by expo prebuild --platform ios
    ├── VidromHome.xcworkspace
    ├── VidromHome/
    │   ├── Info.plist
    │   ├── VidromHome.entitlements
    │   └── ...
    └── Podfile
```

---

## Step 10.13 — New Dependencies Added

```bash
npm install react-native-callkeep react-native-voip-push-notification
npx expo install expo-apple-authentication
```

Updated `package.json` additions:
```json
{
  "dependencies": {
    "expo-apple-authentication": "~7.x.x",
    "react-native-callkeep": "^4.x.x",
    "react-native-voip-push-notification": "^3.x.x"
  }
}
```

---

## Summary of Changes by File

| File | Change | Platform |
|------|--------|----------|
| `app.json` | iOS config, GoogleService-Info.plist, background modes, entitlements, new plugins | iOS |
| `authService.js` | Add `signInWithApple()`, `isAppleSignInAvailable()`, platform-aware `signOut()` | iOS |
| `LoginScreen.js` | Add Apple Sign-In button (iOS only) | iOS |
| `callNotification.js` | Platform switch: CallKit on iOS, Notifee on Android | Both |
| `callkitService.js` | New: CallKit setup, incoming call display, event listeners | iOS |
| `voipService.js` | New: VoIP push registration and handling | iOS |
| `App.js` | Add CallKit initialization on iOS | iOS |
| `HomeScreen.js` | Add CallKit listeners and VoIP push registration on iOS | iOS |
| `GoogleService-Info.plist` | New: Firebase iOS config file | iOS |
| `run-ios.sh` | New: convenience build script | iOS |
| Server `httpRoutes.js` | New endpoint: `/register-voip-token` | Both |
| Server `connectionState.js` | Add `voipTokens` map | Both |

---

## Architecture Notes

### Why CallKit Instead of Local Notifications?

- Apple **requires** VoIP apps to use CallKit for incoming calls (App Store Review Guideline 3.1.7)
- CallKit integrates with the native Phone app — calls appear in Recents, on CarPlay, etc.
- CallKit can display the accepted-UI even when the app is killed (via VoIP push + PushKit)
- Regular iOS local notifications cannot show action buttons on the lock screen the way Android Notifee can

### Why VoIP Push in Addition to FCM?

- FCM on iOS uses standard APNs, which has no guaranteed instant delivery when the app is killed
- VoIP pushes via PushKit are high-priority and guaranteed to wake the app immediately
- Apple requires that every VoIP push results in a CallKit call being reported (otherwise the app is penalized)
- For MVP, FCM alone handles foreground + background. VoIP push adds killed-state reliability.

### Google Sign-In on iOS

- Google Sign-In works on iOS via `@react-native-google-signin/google-signin` — no additional configuration beyond adding the iOS app in Firebase and downloading `GoogleService-Info.plist`
- The `webClientId` used for Android works for iOS as well

### What Stays the Same

- **WebRTC** (`react-native-webrtc`) is fully cross-platform — no iOS-specific changes needed
- **Signaling server** — the FCM push payload already includes `apns` configuration (set up in Step 4)
- **useCallManager.js** — entirely platform-agnostic, works on both iOS and Android
- **IncomingCallScreen.js** — React Native UI, works on both platforms
