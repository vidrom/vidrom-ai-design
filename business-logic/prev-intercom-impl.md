# Previous Vidrom Intercom Implementation

Documentation of all screens and functionality in the previous Android intercom app (`vidrom-intercom`).

## Tech Stack

- **Framework**: React Native with Expo SDK 54
- **Navigation**: React Navigation (Native Stack Navigator)
- **WebRTC**: Via WebView (hidden) pointing to external RTC server pages
- **State Management**: React Context API (`MainContext`)
- **Signaling**: Socket.IO client connecting to `vidrom-rtc.codenroll.co.il`
- **Backend API**: REST API at `https://api.vidrom.com/api`
- **Key Dependencies**: expo-camera, expo-av, expo-brightness, expo-linear-gradient, react-native-webview, socket.io-client, react-native-shadow-2

---

## App Architecture

### Entry Point (`App.js`)
- Prevents the splash screen from auto-hiding using `expo-splash-screen`
- Wraps the entire app in `ContextProvider` for global state
- Waits for context bootstrap to finish (reading from AsyncStorage) before rendering the navigator
- On ready: hides splash screen and renders `AppNavigator`

### Navigator (`AppNavigator.js`)
- Renders `NavigationContainer` with two layers:
  1. **HiddenRTC** (background WebView for passive WebRTC watch/listen) — shown only when no call is in progress
  2. **IntercomRoutes** (full-screen navigation stack) rendered as an overlay

### Routing (`routes.js`)
- **Unauthenticated state** (no environment selected): Only `LoginScreen` is available
- **Authenticated state** (environment selected): Full stack with `WelcomeScreen` as the initial route:
  - `WelcomeScreen` → `SearchScreen` → `CallScreen`
  - `WelcomeScreen` → `AccessCodeScreen`
  - `WelcomeScreen` → `HelpScreen`
  - `WelcomeScreen` → `SettingsScreen`
- Displays `DoorOpenModal` overlay whenever the door-open state is detected

### Global State (`context.js`)
- **`isAuthenticated`** — Whether the device is bound to an environment
- **`environment`** — The selected environment ID (persisted to AsyncStorage as `environmentId`)
- **`intercom`** — Intercom data fetched from API via polling
- **`isDoorOpen`** — Whether the physical door/gate is currently open
- **`callInProgress`** — Whether a call session is active
- **`user`** — User data object
- **Bootstrap**: On app start, loads `environmentId` from AsyncStorage; if found, sets `isAuthenticated = true`
- **Polling**: Polls `GET /intercoms/{id}` every **2 seconds** to check door-open state and update intercom data
- **Door auto-reset**: When door opens, schedules a `PUT /intercoms/{id}` after **3 seconds** to reset `is_door_open` to 0

---

## Screens

### 1. Login Screen (`pages/Login/index.js`)

**Purpose**: Environment/building selection for initial device setup (used by installer/technician).

**Functionality**:
- Fetches all available environments from `GET /environments`
- Displays a scrollable list of environment buttons (showing name or ID)
- On selection:
  - Persists `environmentId` to AsyncStorage
  - Updates global context (`setEnvironment`, `setIsAuthenticated`)
  - App automatically navigates to authenticated flow
- Shows loading spinner while fetching
- Shows error message if API call fails

**UI**: Dark blue background (`#001F3F`), white title "Select Your Intercom Environment", white rounded buttons for each environment option.

---

### 2. Welcome Screen (`pages/Welcome/index.js`)

**Purpose**: Main landing screen / home screen of the intercom. Displayed after authentication.

**Functionality**:
- Fetches the environment details from `GET /environments/{id}` to display the building name
- Shows a welcome message: "Welcome to {buildingName}"
- Displays system connectivity status (green dot + "System is connected" via the Header component)
- Three large navigation buttons in a vertical column:
  1. **Apartment Search** → navigates to `SearchScreen` with `searchType: 'name'`
  2. **Enter Access Code** → navigates to `AccessCodeScreen`
  3. **Help** → navigates to `HelpScreen`

**UI**: Gradient background, Vidrom logo at top, large rounded card-style buttons (240px height) with shadow effects, icons (search icon, key icon, help icon) alongside text labels.

---

### 3. Search Screen (`pages/Search/index.js`)

**Purpose**: Allows visitors to search for a resident/unit to call.

**Functionality**:
- Fetches all units for the current environment from `GET /units?environment_id={id}`
- Deduplicates units by `unit_number`, sorts numerically
- Text input for searching by name or apartment number
- Toggle button to switch between name keyboard (alphabetic) and apartment number keyboard (numeric)
- Real-time filtering as the user types (matches `apartment_title` or `unit_number`)
- Search results displayed in a `FlatList` showing apartment title and unit number
- Tapping a result navigates to `CallScreen` with the selected unit data

**UI**: Header with title ("Search by name" / "Search by apartment"), rounded search input field with toggle icon (dialpad/keyboard), results list with semi-transparent background and rounded corners.

---

### 4. Access Code Screen (`pages/AccessCode/index.js`)

**Purpose**: Allows visitors to enter a 4-digit access code to open the gate.

**Functionality**:
- Four individual digit input fields (numeric, secure entry / masked)
- Auto-focus on first digit on mount
- Auto-advance to next digit as each is entered
- Backspace handling to return to previous digit
- Auto-submits when 4th digit is entered
- Validates code against hardcoded value `"1234"`:
  - **Success**: Shows `AccessCodeModal` with green background and "Access Granted / The gate is opening." message for 2.5 seconds, then navigates back to `WelcomeScreen`
  - **Failure**: Shows `AccessCodeModal` with red background and "Access Denied / Incorrect Code." message for 2.5 seconds, then clears all digits and refocuses first input

**UI**: Large square digit inputs (70×96px) with white background, rounded corners, centered in a row.

---

### 5. Call Screen (`pages/Call/index.js`)

**Purpose**: Video/audio call interface for a visitor calling a resident.

**Functionality**:
- Receives selected unit data via navigation params
- On mount, sends call request via `POST /calls` with `environment_id` and `unit_id`
- Constructs WebRTC room name: `env{environment_id}_unit{unitId}`
- Loads a hidden WebView pointing to the caller RTC page: `https://rtc.vidrom.com/call/caller.html?room={room}&touch=1`
- **Call status polling**: Every 3 seconds, checks `GET /calls/{callId}` for status updates
- **No-answer timeout**: If no answer within **30 seconds**, auto-ends the call
- **Call states**:
  - `calling` — Shows ringing UI with phone icon, "Calling {name} (Apt. {number}) ..." text, and red End Call button
  - `accepted` — Shows connected UI with call duration timer (HH:MM:SS format), resident info, and red End Call button
  - `ended` — Navigates back to WelcomeScreen
- **End call**:
  - Injects `endCall()` into the WebView
  - Sends `PUT /calls/{callId}` with `status: 'ended'`
  - Navigates to WelcomeScreen
- Handles `end-call` messages from WebView (remote hang-up)

**UI**: Background image (`callBg.png`), Vidrom logo, centered call icon in a large white circle (168px), call status text, large red end-call button (90px circle) at the bottom.

---

### 6. Help Screen (`pages/Help/index.js`)

**Purpose**: FAQ / Help section with expandable Q&A items.

**Functionality**:
- Displays a list of 9 predefined Q&A items in a `FlatList`
- Tap a question to expand/collapse its answer (accordion style)
- Only one answer visible at a time (tapping another collapses the current)
- Auto-scrolls to center the selected question in view

**Q&A Topics**:
1. How do I reset my password?
2. How can I contact support?
3. Where can I update my profile?
4. What are the benefits of two-factor authentication?
5. How do I subscribe to your newsletter?
6. Can I change my subscription plan at any time?
7. What steps can I take to secure my account?
8. How do I unsubscribe from emails?
9. How can I change my email preferences?

**UI**: Semi-transparent list container with rounded corners, bold question text, expandable white answer panels.

---

### 7. Settings Screen (`pages/Settings/index.js`)

**Purpose**: Device settings and logout/reset functionality.

**Functionality**:
- Displays current user info:
  - User Name
  - User ID
  - Environment ID
- **Screen Brightness** slider (0–1) using `expo-brightness`
- **Volume** slider (0–1) using `expo-av` Audio API
- **Log Out** button (acts as a full device reset):
  - Clears `user` from AsyncStorage
  - Clears `environmentId` from AsyncStorage
  - Resets context state (`user`, `environment`, `isAuthenticated`)
  - Returns device to Login/environment selection screen

**UI**: Semi-transparent row items for user info, green-themed sliders, red "Log Out" button.

---

## Shared Components

### Header (`components/Header/index.js`)
- Displays Vidrom logo (touchable but no-op)
- Title text (uppercase, bold, white)
- Two modes controlled by `status` prop:
  - `status=true`: Shows connectivity indicator (green/red dot + text), polls `GET /environments/{id}` every 10 seconds and on app resume
  - `status=false`: Shows a back navigation button (arrow icon)

### Bg (Background Gradient) (`components/Bg/index.js`)
- Full-screen `LinearGradient` wrapper (dark blue to near-black diagonal gradient)
- Used as the background on most screens

### HiddenRTC (`components/HiddenRTC.js`)
- Background WebRTC watcher component
- Renders a 1×1 pixel invisible WebView pointing to `https://rtc.vidrom.com/watch/index.html?userId=user{environment}`
- Handles camera/microphone permission checking and requesting
- Shows permission denied UI with "Try Again" and "Open Settings" buttons if needed
- Shows error/retry UI if WebView fails to load
- Only rendered when no call is in progress

### AccessCodeModal (`components/modals/AccessCodeModal/index.js`)
- Full-screen modal overlay with semi-transparent dark background
- Success state: Green background, check-circle icon, custom message
- Failure state: Red background, cancel icon, custom message
- Animated fade transition

### DoorOpenModal (`components/modals/DoorOpenModal/index.js`)
- Full-width banner at the top of the screen (z-index 1000)
- Green background with white bold text "Gate is open"
- Shown globally whenever `isDoorOpen` is true in context
- Auto-dismissed after 3 seconds when the door reset API call completes

---

## API Endpoints Used

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/environments` | Fetch list of all available environments (Login screen) |
| GET | `/environments/{id}` | Fetch single environment details (Welcome, Header connectivity check) |
| GET | `/units?environment_id={id}` | Fetch all units/apartments for an environment (Search screen) |
| GET | `/intercoms/{id}` | Poll intercom status including door state (Context polling every 2s) |
| PUT | `/intercoms/{id}` | Reset door-open state after 3s timeout (Context) |
| POST | `/calls` | Initiate a new call (Call screen) |
| GET | `/calls/{id}` | Poll call status (Call screen, every 3s) |
| PUT | `/calls/{id}` | Update call status to 'ended' (Call screen) |

---

## WebRTC Architecture

- Uses **WebView-based WebRTC** (not native)
- Two RTC modes:
  1. **Watch mode** (passive): `https://rtc.vidrom.com/watch/index.html?userId=user{env}` — runs in hidden WebView when idle
  2. **Call mode** (active): `https://rtc.vidrom.com/call/caller.html?room={room}&touch=1` — runs during active calls
- Socket.IO signaling via `vidrom-rtc.codenroll.co.il` (configured in `_socket.js` but WebView handles its own signaling)

---

## Navigation Flow

```
App Start
  │
  ├─ No saved environmentId → LoginScreen (environment selection)
  │                              │
  │                              └─ Select environment → saves to AsyncStorage → enters authenticated flow
  │
  └─ Has saved environmentId → Authenticated Flow
                                  │
                                  ├─ WelcomeScreen (home)
                                  │     ├─ "Apartment Search" → SearchScreen
                                  │     │                          └─ Tap unit → CallScreen
                                  │     ├─ "Enter Access Code" → AccessCodeScreen
                                  │     │                          ├─ Correct → modal + back to Welcome
                                  │     │                          └─ Wrong → modal + retry
                                  │     └─ "Help" → HelpScreen
                                  │
                                  └─ SettingsScreen (accessible from Welcome via Header/nav)
                                        └─ "Log Out" → clears storage → LoginScreen
```

---

## Assets

- `Logo.png` — Vidrom logo (displayed in Header)
- `callBg.png` — Background image for Call screen
- `splash.png` — Splash screen image
- `icon.png` / `adaptive-icon.png` / `favicon.png` — App icons
- `svg.js` — SVG icon components (`IconSearch`, `IconKey`) used on Welcome screen
