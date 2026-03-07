# Previous Vidrom Home App Implementation

Documentation of all screens and functionality in the previous Android/iOS home (resident) app (`vidrom_home`).

## Tech Stack

- **Framework**: React Native with Expo SDK 54
- **Navigation**: React Navigation (Native Stack Navigator)
- **WebRTC**: Via WebView (hidden) pointing to external RTC server pages
- **State Management**: React Context API (`MainContext`)
- **Authentication**: Google Sign-In (`@react-native-google-signin/google-signin`) + Apple Sign-In (`expo-apple-authentication`)
- **Push Notifications**: Expo Notifications (Expo Push Token)
- **Backend API**: REST API at `https://api.vidrom.com/api`
- **RTC Server**: `https://rtc.vidrom.com` (call/receiver and watch pages)
- **Key Dependencies**: expo-camera, expo-notifications, expo-apple-authentication, react-native-webview, react-native-scroll-indicator, react-native-svg

---

## App Architecture

### Entry Point (`App.js` → `Layout.js`)
- `App.js` simply renders `Layout`
- `Layout.js` wraps the app in `NavigationContainer` → `ContextProvider` → `AppRoutes`

### Global State (`context.js`)
- **`user`** — User object (fetched from AsyncStorage on boot)
- **`userToken`** — JWT auth token (stored in AsyncStorage as raw string)
- **`unit`** — Selected apartment/unit object
- **`environment`** — Environment (building) data fetched from API
- **`environment_id`** — Environment ID
- **`intercoms`** — Array of intercoms for the environment
- **`darkMode`** — Dark/light theme preference (persisted to AsyncStorage)
- **`sleepMode`** — Sleep mode toggle (persisted to AsyncStorage)
- **`isDoorOpen`** — Whether the physical door/gate is currently open
- **`incomingCall`** — Current incoming call object (or null)
- **`logoBig`** — Logo size preference

**Bootstrap flow** (on mount):
1. Load `userToken` and `userData` from AsyncStorage
2. Load `environmentID` from AsyncStorage → fetch environment from API → fetch intercoms
3. Load `unit` from AsyncStorage → fetch unit from API
4. Load `darkMode` and `sleepMode` preferences from AsyncStorage
5. Set `ready = true`
6. Navigate based on state:
   - No user → `Login`
   - User but no unit → `ApartmentSelection`
   - Both → `Home`

### Routing (`routes.js`)
- **Initial route**: Determined by auth state (`Login` / `ApartmentSelection` / `Home`)
- **All screens** (registered in the stack):
  - `Login`, `Home`, `ApartmentSelection`, `Conversation`, `Activity`, `Notifications`, `Menu`, `Watch`, `Support`, `About`
- **Bottom Navigation** shown on: Home, ApartmentSelection, Watch, Menu, Activity, Notifications, Support, About
- **Push notification handler**: Listens for expo-notifications, navigates to Conversation if there's an incoming call
- **Incoming call polling**: Every **3 seconds**, polls `GET /calls?environment_id={env}&unit_id={unit}` for calls with status `calling`
- **Global overlays**:
  - `DoorOpenModal` — shown when `isDoorOpen` is true
  - `CallModal` — shown when `incomingCall` is not null

### Auth Service (`services/auth.js`)
- **`loginWithGoogle(userInfo)`** — Sends Google `idToken` to `POST /users/auth` with `method: "google"`
- **`loginWithApple(identityToken)`** — Sends Apple `identityToken` to `POST /users/auth` with `method: "apple"`
- Returns session token + user data on success

### Utility Functions (`functions.js`)
- **`apiRequest(method, endpoint, body)`** — Generic fetch with auto-attached `Authorization: Bearer` header from AsyncStorage
- **`GetDataLocal(key)`** — Read from AsyncStorage (handles token as raw string, everything else as JSON)
- **`StoreDataLocal(key, value)`** — Write to AsyncStorage
- **`RemoveDataLocal(key)`** — Delete from AsyncStorage
- **`prettifyDate(date)`** — Format date with Intl.DateTimeFormat
- **`registerForPushNotifications(userId)`** — Request notification permissions, get Expo push token, save to `POST /users/{userId}/expo-token`

---

## Screens

### 1. Login Screen (`pages/Login/index.js`)

**Purpose**: User authentication via Google or Apple sign-in.

**Functionality**:
- Configures Google Sign-In with web client ID and iOS client ID on mount
- **Google Sign-In**:
  - Checks Play Services availability (Android)
  - Signs in via `GoogleSignin.signIn()`
  - Sends ID token to backend via `loginWithGoogle()`
  - On success: stores token + user data in AsyncStorage, requests camera/mic permissions, registers for push notifications
- **Apple Sign-In** (iOS only):
  - Requests full name + email scopes via `AppleAuthentication.signInAsync()`
  - Sends identity token to backend via `loginWithApple()`
  - Handles 404 (account not found) with specific error message
  - On success: same storage + permissions + push notification flow as Google
- Shows loading spinner during auth process
- Shows alert dialogs on failure

**Sub-components** (from previous implementation, now partially unused):
- `LoginForm` — Email/phone input with keyboard type toggle (email/numeric), send button
- `CodeForm` — Numeric code verification input with SVG send button
- `LoginHeader` — Logo + "Sign in to Vidrom" + subtitle

**UI**: App background image, "Welcome to Vidrom" title, black "Sign in with Google" button, Apple Sign-In button (iOS only).

---

### 2. Apartment Selection Screen (`pages/ApartmentSelection/index.js`)

**Purpose**: Select which apartment/unit the user belongs to after login.

**Functionality**:
- Fetches user's units from `GET /units?user_id={userId}`
- Sets `environment_id` from the user's `environment_id` field
- **Auto-selection**: If user has only 1 unit, auto-selects it immediately
- On unit selection:
  - Stores unit data to AsyncStorage (`unit` key)
  - Stores `environmentID` to AsyncStorage
  - Updates context `setUnit()` → triggers navigation to Home
- Shows loading spinner while fetching
- Shows "No apartments found." if no units available

**UI**: Logo at top, "Select apartment" title, scrollable list of apartment cards showing apartment title + unit number with right-arrow chevron. Dark mode support with different card styling.

---

### 3. Home Screen (`pages/Home/index.js`)

**Purpose**: Main dashboard for the authenticated resident.

**Functionality**:
- Fetches building address from `GET /environments/{id}`
- Fetches latest 5 notifications from `GET /notifications` (sorted by date descending)
- Checks and requests camera/microphone permissions on mount
- **Open Gate** button:
  1. Immediately shows `DoorOpenModal` for responsive UI
  2. Fetches intercom data via `GET /intercoms/{intercom_id}`
  3. Sends `PUT /intercoms/{intercom_id}` with `is_door_open: 1`
  4. Logs the action via `POST /logs`
- **Watch Camera** button → navigates to Watch screen
- Shows permission-denied UI with "Open device settings" link if camera/mic not granted
- Displays "Important updates" section with latest 5 notifications

**UI**: App background, logo, "Welcome home" heading, "apartment {number}, {address}" subtitle, two primary action buttons side-by-side (Open gate for guest + Watch camera), notification cards below. Dark mode support.

---

### 4. Conversation Screen (`pages/Conversation/index.js`)

**Purpose**: Active video call with the intercom visitor (answering an incoming call).

**Functionality**:
- Receives `offer` (call data) via navigation params
- Dismisses all notifications and clears `incomingCall` on mount
- Ensures camera/mic permissions
- Sends `PUT /calls/{id}` with `status: 'accepted'` on mount
- Loads WebView pointing to `https://rtc.vidrom.com/call/receiver.html?room=env{env}_unit{unit}`
- **Injected JavaScript**: Extensive WebRTC debugging instrumentation:
  - Wraps `getUserMedia`, `RTCPeerConnection`, `addTrack`, `setLocalDescription`
  - Logs all ICE connection state changes, signaling state, track events
  - Periodically reports outbound-rtp stats (every 4s)
- **Call status polling**: Every 3 seconds, checks `GET /calls/{id}` — if `ended`, navigates to Home
- **Call timeout**: Auto-ends call after **60 seconds**
- **End Call** button:
  - Injects `endCall()` into WebView
  - Sends `PUT /calls/{id}` with `status: 'ended'`
  - Resets navigation to Home
- **Open Gate** button (during call):
  - Same logic as Home's Open Gate
  - Shows `DoorOpenModal`, updates intercom, logs action

**UI**: Full-screen WebView (video feed), close button (X) in top-right with semi-transparent background, red "Open the gate" button at bottom center.

---

### 5. Watch Screen (`pages/WatchScreen/index.js`)

**Purpose**: Live camera view from the intercom (on-demand viewing, no active call).

**Functionality**:
- Loads WebView pointing to `https://rtc.vidrom.com/watch/index.html?intercomId=user{intercom_id}`
- Injected JavaScript: Wraps `console.log` for debugging, instruments `RTCPeerConnection` track events
- **Open Gate** button:
  - Fetches intercom data via `GET /intercoms/{intercom_id}`
  - Sends `PUT /intercoms/{id}` with `is_door_open: 1`
  - Toggles `isDoorOpen` state
  - Auto-resets door after **5 seconds** via `PUT /intercoms/{id}` with `is_door_open: 0`
- Close button: Injects `stopWatcher()` into WebView + navigates back
- Displays environment address and gate ID overlay
- Cleans up by calling `stopWatcher()` on unmount

**UI**: Full-screen WebView (camera feed), close button top-left, environment label (address + gate ID) centered at top, "Open Gate" button at bottom center.

---

### 6. Activity Screen (`pages/ActivityScreen/index.js`)

**Purpose**: View log of all building/intercom activity.

**Functionality**:
- Fetches all logs from `GET /logs` on mount
- Displays logs in reverse chronological order
- Each log shows: log text + formatted date/time
- Custom scroll indicator via `react-native-scroll-indicator`

**UI**: "Activity" title, scrollable list of log cards with text and timestamp. Dark mode support (different card backgrounds).

---

### 7. Notifications Screen (`pages/NotificationsScreen/index.js`)

**Purpose**: View all building/management notifications.

**Functionality**:
- Fetches all notifications from `GET /notifications` on mount
- Displays each notification with note text + formatted date/time
- Shows "No notifications" when empty
- Custom scroll indicator via `react-native-scroll-indicator`

**UI**: "Notifications" title, scrollable list of notification cards with white backgrounds. Dark mode support.

---

### 8. Menu Screen (`pages/MenuScreen/index.js`)

**Purpose**: Settings, preferences, and navigation hub.

**Functionality**:
- Fetches user's units from `GET /units?user_id={id}` to determine if "Change Apartment" should be shown
- Menu items (rendered as rows with icons):
  1. **Change Apartment** (only shown if user has > 1 unit) — clears unit, navigates to ApartmentSelection. Shows current apartment name on the right
  2. **Dark Mode** — toggles dark/light theme
  3. **Change Ring** — opens device notification settings (via `Linking.openSettings()`)
  4. **Sleep** — toggle switch for sleep mode (persisted to AsyncStorage)
  5. **Support** — navigates to Support screen
  6. **About** — navigates to About screen
  7. **Log Out** — clears all AsyncStorage data, resets context, navigates to Login

**UI**: "Menu" title, scrollable list of menu rows with icons + labels + optional right-side controls (text badges, switches). Dark mode support.

---

### 9. Support Screen (`pages/Support/index.js`)

**Purpose**: FAQ and manager contact.

**Functionality**:
- Displays 3 Q&A items in expandable accordion format:
  1. How do I change my notification settings?
  2. How can I reset my password?
  3. Who do I contact for technical issues?
- **Call Manager** button — opens phone dialer via `tel:+123456789`

**UI**: "Support" title, expandable Q&A cards, "Call Manager" button at bottom.

---

### 10. About Screen (`pages/About/index.js`)

**Purpose**: App information placeholder.

**Functionality**:
- Displays "About Vidrom Home" title
- Placeholder text: "Here will come a grateful text about the company and the application."

**UI**: Simple scrollable view with title and centered placeholder text.

---

## Shared Components

### AppBackground (`components/AppBackground/index.js`)
- Full-screen `ImageBackground` wrapper
- Switches between `background-light.jpg` (light mode) and `background-dark.jpg` (dark mode)

### Logo (`components/Logo/index.js`)
- Displays Vidrom logo (centered, 120×32)
- Switches between `logo-black.png` (light mode) and `Logo.png` (dark mode)

### Title (`components/Title/index.js`)
- Reusable text title component
- Supports `big` prop for larger font size
- Responsive: larger fonts on wider screens (> 400px)
- Dark mode text color support

### Button (`components/Button/index.js`)
- Reusable button with `label`, `action`, and `format` ('normal' or 'inverted') props
- Dark mode color inversion support
- Rounded corners (12px), full-width, centered text

### BottomNav (`components/BottomNav/index.js`)
- Fixed bottom navigation bar with 4 tabs:
  1. **Home** (home-outline icon)
  2. **Activity** (list-circle-outline icon)
  3. **Notifications** (notifications-outline icon)
  4. **Menu** (menu-outline icon)
- Watch tab is commented out (disabled)
- Active tab highlighted with inverted colors
- Responsive icon sizes (42px on > 400px screens, 32px otherwise)
- Dark mode support

### CallModal (`components/modals/CallModal/index.js`)
- Animated slide-in modal from bottom
- Shows "Incoming request at the entrance" + "A guest is requesting access"
- Door-open icon on the left
- Tap to accept the call (slides out, triggers `onAccept`)
- Used globally in routes when `incomingCall` is not null

### DoorOpenModal (`components/modals/DoorOpenModal/index.js`)
- Animated slide-in modal from bottom
- Shows "The Door is Open" with door-open icon
- Auto-slides out after **5 seconds**
- On dismiss: sends `PUT /intercoms/{id}` to reset `is_door_open: 0`
- Tap to dismiss early

---

## API Endpoints Used

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/users/auth` | Authenticate via Google/Apple (exchange idToken for session) |
| POST | `/users/{id}/expo-token` | Register push notification token |
| GET | `/environments/{id}` | Fetch environment/building details |
| GET | `/units?user_id={id}` | Fetch user's assigned apartments |
| GET | `/units/{id}` | Fetch single unit details |
| GET | `/intercoms/environment/{id}` | Fetch intercoms for an environment |
| GET | `/intercoms/{id}` | Fetch single intercom data |
| PUT | `/intercoms/{id}` | Update intercom state (door open/close) |
| GET | `/calls?environment_id={id}&unit_id={id}` | Poll for incoming calls |
| PUT | `/calls/{id}` | Update call status (accepted/ended/rejected) |
| GET | `/notifications` | Fetch all notifications |
| GET | `/logs` | Fetch all activity logs |
| POST | `/logs` | Create a new log entry (e.g., door opened) |

---

## WebRTC Architecture

- Uses **WebView-based WebRTC** (not native)
- Two RTC modes:
  1. **Call mode** (answering): `https://rtc.vidrom.com/call/receiver.html?room=env{env}_unit{unit}` — active call with intercom
  2. **Watch mode** (on-demand): `https://rtc.vidrom.com/watch/index.html?intercomId=user{intercom_id}` — passive camera viewing
- Extensive injected JavaScript for debugging (getUserMedia wrapping, RTCPeerConnection instrumentation, periodic stats reporting)

---

## Push Notifications

- Uses **Expo Push Notifications** (Expo Push Token)
- Registered on login via `registerForPushNotifications(userId)`
- Token saved to backend via `POST /users/{userId}/expo-token`
- Notification channel configured for Android (max importance)
- Notification handler configured to show alert + play sound
- Notification tap listener: navigates to Conversation screen with current incoming call data

---

## Navigation Flow

```
App Start → Bootstrap (load from AsyncStorage)
  │
  ├─ No userToken → Login
  │                   ├─ Google Sign-In → auth → store token → ApartmentSelection (or Home if unit exists)
  │                   └─ Apple Sign-In → auth → store token → ApartmentSelection (or Home if unit exists)
  │
  ├─ Has token, no unit → ApartmentSelection
  │                         ├─ 1 unit → auto-select → Home
  │                         └─ Multiple units → pick one → Home
  │
  └─ Has token + unit → Home
                          │
                          ├─ "Open gate for guest" → DoorOpenModal (5s) + API update
                          ├─ "Watch camera" → WatchScreen (live WebRTC view)
                          │                     └─ "Open Gate" → API update + auto-reset 5s
                          │
                          ├─ [Incoming call detected] → CallModal slide-in
                          │     └─ Tap to accept → Conversation (WebRTC call)
                          │                           ├─ "Open the gate" → DoorOpenModal + API
                          │                           ├─ End call → API update → Home
                          │                           └─ Auto-end after 60s
                          │
                          ├─ Bottom Nav: Activity → logs list
                          ├─ Bottom Nav: Notifications → notifications list
                          └─ Bottom Nav: Menu
                                ├─ Change Apartment → ApartmentSelection
                                ├─ Dark Mode toggle
                                ├─ Change Ring → device settings
                                ├─ Sleep toggle
                                ├─ Support → FAQ + Call Manager
                                ├─ About → placeholder
                                └─ Log Out → clear storage → Login
```

---

## Dark Mode

- Persisted to AsyncStorage as `darkMode` boolean
- Toggled from Menu screen
- Affects:
  - Background images (light/dark variants)
  - Logo (black/white variants)
  - Text colors throughout
  - Bottom nav colors (inverted)
  - Card/container backgrounds
  - Button colors
  - Status bar style

---

## Assets

- `Logo.png` / `logo-black.png` / `Logo.svg` — Vidrom logos (dark/light/vector)
- `vidrom-logo.png` — Additional logo variant
- `background-light.jpg` / `background-dark.jpg` — Full-screen background images (light/dark mode)
- `callBg.png` — Call screen background
- `splash.png` / `splash-icon.png` — Splash screen assets
- `icon.png` / `adaptive-icon.png` / `favicon.png` — App icons
- `svg.js` — SVG icon components
