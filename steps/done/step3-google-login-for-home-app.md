# Step 3 - Google Login for Home App

## Goal

Add Google Sign-In authentication to the vidrom-home app, allowing residents to log in with their Google account. This establishes user identity before setting up notifications.

## Overview

- Use Firebase Authentication with Google Sign-In provider
- Display login screen on app start if not authenticated
- Store user session and display user info
- Prepare for linking FCM tokens to user accounts in later steps

## Prerequisites

1. Firebase project created at [Firebase Console](https://console.firebase.google.com/)
2. vidrom-home app registered in Firebase (Android package: `com.vidrom.ai.home`)

---

## Step 3.1 - Firebase Project Setup

1. Create a new Firebase project (or use existing) named "Vidrom"
2. Add Android app for vidrom-home:
   - Package name: `com.vidrom.ai.home`
   - Download `google-services.json` → place in `vidrom-ai-home/android/app/`
3. Enable Google Sign-In provider:
   - Go to Firebase Console → Authentication → Sign-in method
   - Enable "Google" provider
   - Note the Web Client ID (needed for Android configuration)
4. (iOS) Add iOS app with bundle ID and download `GoogleService-Info.plist`

---

## Step 3.2 - Install Firebase Auth Dependencies

1. Install required packages:
   ```bash
   cd vidrom-ai-home
   npx expo install @react-native-firebase/app @react-native-firebase/auth
   npm install @react-native-google-signin/google-signin
   ```

2. Update `app.json` to include Firebase and Google Sign-In plugins:
   ```json
   {
     "expo": {
       "plugins": [
         "@react-native-firebase/app",
         "@react-native-firebase/auth",
         [
           "@react-native-google-signin/google-signin",
           {
             "iosUrlScheme": "com.googleusercontent.apps.YOUR_IOS_CLIENT_ID"
           }
         ]
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

## Step 3.3 - Configure Google Sign-In

1. Get the Web Client ID from Firebase Console:
   - Go to Project Settings → General
   - Under "Your apps", find the Web Client ID
   - Or go to Google Cloud Console → APIs & Services → Credentials

2. Create `authService.js` in vidrom-home:
   ```javascript
   import auth from '@react-native-firebase/auth';
   import { GoogleSignin } from '@react-native-google-signin/google-signin';
   
   // Configure Google Sign-In (call once on app start)
   export function configureGoogleSignIn() {
     GoogleSignin.configure({
       webClientId: 'YOUR_WEB_CLIENT_ID.apps.googleusercontent.com',
     });
   }
   
   // Sign in with Google
   export async function signInWithGoogle() {
     // Check if device supports Google Play
     await GoogleSignin.hasPlayServices({ showPlayServicesUpdateDialog: true });
     
     // Get the user's ID token
     const { idToken } = await GoogleSignin.signIn();
     
     // Create a Google credential with the token
     const googleCredential = auth.GoogleAuthProvider.credential(idToken);
     
     // Sign-in the user with the credential
     return auth().signInWithCredential(googleCredential);
   }
   
   // Sign out
   export async function signOut() {
     await GoogleSignin.revokeAccess();
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

---

## Step 3.4 - Create Login Screen UI

1. Create `LoginScreen.js` in vidrom-home:
   ```javascript
   import React, { useState } from 'react';
   import {
     View,
     Text,
     TouchableOpacity,
     StyleSheet,
     ActivityIndicator,
     Alert,
   } from 'react-native';
   import { signInWithGoogle } from './authService';
   
   export default function LoginScreen() {
     const [loading, setLoading] = useState(false);
   
     const handleGoogleSignIn = async () => {
       setLoading(true);
       try {
         await signInWithGoogle();
         // Auth state listener in App.js will handle navigation
       } catch (error) {
         console.error('Google Sign-In error:', error);
         Alert.alert('Sign-In Error', error.message);
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
       </View>
     );
   }
   
   const styles = StyleSheet.create({
     container: {
       flex: 1,
       justifyContent: 'center',
       alignItems: 'center',
       backgroundColor: '#f5f5f5',
       padding: 20,
     },
     title: {
       fontSize: 32,
       fontWeight: 'bold',
       marginBottom: 10,
     },
     subtitle: {
       fontSize: 16,
       color: '#666',
       marginBottom: 40,
     },
     googleButton: {
       backgroundColor: '#4285F4',
       paddingVertical: 15,
       paddingHorizontal: 30,
       borderRadius: 8,
       minWidth: 250,
       alignItems: 'center',
     },
     buttonText: {
       color: '#fff',
       fontSize: 16,
       fontWeight: '600',
     },
   });
   ```

---

## Step 3.5 - Update App.js for Auth Flow

1. Update `App.js` to handle authentication state:
   ```javascript
   import React, { useEffect, useState } from 'react';
   import { configureGoogleSignIn, onAuthStateChanged, signOut } from './authService';
   import LoginScreen from './LoginScreen';
   
   export default function App() {
     const [user, setUser] = useState(null);
     const [initializing, setInitializing] = useState(true);
   
     useEffect(() => {
       // Configure Google Sign-In once
       configureGoogleSignIn();
       
       // Listen to auth state changes
       const unsubscribe = onAuthStateChanged((user) => {
         setUser(user);
         if (initializing) setInitializing(false);
       });
   
       return unsubscribe;
     }, []);
   
     if (initializing) {
       return null; // Or splash screen
     }
   
     // Show login screen if not authenticated
     if (!user) {
       return <LoginScreen />;
     }
   
     // Show main app (existing call UI) if authenticated
     return (
       // ... existing app content with user info available
       // user.displayName, user.email, user.photoURL
     );
   }
   ```

2. Add sign out button to the main app UI:
   ```javascript
   <TouchableOpacity onPress={signOut} style={styles.signOutButton}>
     <Text>Sign Out ({user.email})</Text>
   </TouchableOpacity>
   ```

---

## Step 3.6 - Testing Checklist

1. **Fresh Install Test:**
   - Uninstall app, reinstall
   - Verify login screen appears

2. **Google Sign-In Test:**
   - Tap "Sign in with Google"
   - Select Google account
   - Verify successful login and main app appears

3. **Persistence Test:**
   - Close app completely
   - Reopen app
   - Verify user is still logged in (no login screen)

4. **Sign Out Test:**
   - Tap sign out button
   - Verify login screen appears
   - Verify can sign in again

---

## Step 3.7 - Project Structure After Completion

```
vidrom-ai/
├── vidrom-ai-home/
│   ├── App.js              # Updated with auth flow
│   ├── authService.js      # New: Google Sign-In logic
│   ├── LoginScreen.js      # New: Login UI
│   └── android/
│       └── app/
│           └── google-services.json
```

---

## Next Step

See [step4.md](step4.md) for push notifications implementation (FCM tokens will be linked to authenticated users).
