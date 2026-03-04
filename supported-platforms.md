# Supported Platforms — Vidrom Home App

## Android

| Setting | Value |
|---------|-------|
| **Minimum SDK** | API 24 (Android 7.0 Nougat) |
| **Target SDK** | API 35 (Android 15) |
| **Architecture** | New Architecture enabled (`newArchEnabled: true`) |

### Android Version Coverage

| Android Version | API Level | Status | Notes |
|----------------|-----------|--------|-------|
| Android 15 | API 35 | Supported | Latest, target SDK |
| Android 14 | API 34 | Supported | |
| Android 13 | API 33 | Supported | |
| Android 12L | API 32 | Supported | |
| Android 12 | API 31 | Supported | |
| Android 11 | API 30 | Supported | |
| Android 10 | API 29 | Supported | |
| Android 9 (Pie) | API 28 | Supported | |
| Android 8.1 (Oreo) | API 27 | Supported | |
| Android 8.0 (Oreo) | API 26 | Supported | |
| Android 7.1 (Nougat) | API 25 | Supported | |
| Android 7.0 (Nougat) | API 24 | Supported | Minimum supported |
| Android 6.0 and below | API 23- | Not supported | Below minSdkVersion |

> **Why API 24?** React Native WebRTC requires API 24+ for reliable media stream handling. Expo SDK 54 also lists API 24 as the minimum.

### Android-Specific Features

- **Full-screen incoming call notification** via Notifee (`USE_FULL_SCREEN_INTENT`)
- **Google Sign-In** via `@react-native-google-signin/google-signin`
- **FCM push notifications** via `@react-native-firebase/messaging`
- **WebRTC** via `react-native-webrtc`

---

## iOS

| Setting | Value |
|---------|-------|
| **Minimum Deployment Target** | iOS 15.0 |
| **Architecture** | New Architecture enabled |
| **Bundle Identifier** | `com.vidrom.ai.home` |

### iOS Version Coverage

| iOS Version | Status | Notes |
|-------------|--------|-------|
| iOS 18.x | Supported | Latest |
| iOS 17.x | Supported | |
| iOS 16.x | Supported | |
| iOS 15.x | Supported | Minimum supported |
| iOS 14.x and below | Not supported | Below deployment target |

### Supported iPhone Models (iOS 15+)

| Model | Earliest Supported iOS |
|-------|----------------------|
| iPhone 16 / 16 Pro / 16 Pro Max | iOS 18 |
| iPhone 15 / 15 Pro / 15 Pro Max | iOS 17 |
| iPhone 14 / 14 Pro / 14 Pro Max | iOS 16 |
| iPhone 13 / 13 Pro / 13 Pro Max / 13 mini | iOS 15 |
| iPhone 12 / 12 Pro / 12 Pro Max / 12 mini | iOS 15 |
| iPhone 11 / 11 Pro / 11 Pro Max | iOS 15 |
| iPhone XS / XS Max | iOS 15 |
| iPhone XR | iOS 15 |
| iPhone X | iOS 15 |
| iPhone 8 / 8 Plus | iOS 15 |
| iPhone SE (2nd gen, 2020) | iOS 15 |
| iPhone SE (3rd gen, 2022) | iOS 15 |
| iPhone 7 / 7 Plus | iOS 15 |
| iPhone 6s / 6s Plus | iOS 15 (max iOS 15.8) |
| iPhone SE (1st gen) | iOS 15 (max iOS 15.8) |
| iPhone 6 and older | Not supported | Cannot run iOS 15 |

> **Why iOS 15?** Expo SDK 54 requires iOS 15.0+. Apple Sign-In is available on iOS 13+, and CallKit has been available since iOS 10, so the iOS 15 floor covers both.

### iOS-Specific Features

- **Apple Sign-In** via `expo-apple-authentication`
- **Google Sign-In** via `@react-native-google-signin/google-signin`
- **CallKit incoming call UI** via `react-native-callkeep`
- **VoIP push notifications** via `react-native-voip-push-notification` (PushKit)
- **FCM push notifications** via `@react-native-firebase/messaging` (uses APNs under the hood)
- **WebRTC** via `react-native-webrtc`

---

## Build Stack

| Component | Version |
|-----------|---------|
| Expo SDK | 54 |
| React Native | 0.81.5 |
| React | 19.1.0 |
| Firebase | 23.8.6 (`@react-native-firebase/*`) |
| Notifee | 9.1.8 (Android only) |
| WebRTC | 124.0.7 |

---

## Market Coverage Estimate

- **Android API 24+** covers ~97% of active Android devices (as of 2026)
- **iOS 15+** covers ~95%+ of active iPhones (as of 2026)
