# Step 1 - Basic Expo App Scaffolding

## Goal

Create two minimal Expo mobile apps ("vidrom-intercom" and "vidrom-home"), each displaying a single page with a "Vidrom Intercom" label.

## Overview

- vidrom-intercom mobile app using Expo
  - Single page UI
  - Has a label with "Vidrom Intercom" text displayed

- vidrom-home mobile app using Expo
  - Single page UI
  - Has a label with "Vidrom Home" text displayed

## Prerequisites

1. Install Node.js (LTS version, 20+) - required by latest Expo SDK / Metro
   - Recommended: use [nvm-windows](https://github.com/coreybutler/nvm-windows)
2. Install EAS CLI globally: `npm install -g eas-cli`
3. Log in to Expo account: `npx expo login` (use vidromintercom@gmail.com)

**Note:** The modern Expo local CLI is used via `npx expo` (bundled with each project). The legacy global `expo-cli` package is deprecated and NOT required.

## Step 1.1 - Create the "vidrom-intercom" Expo App

1. From the project root, run:
   ```bash
   npx create-expo-app vidrom-intercom --template blank
   ```

2. Navigate into the project:
   ```bash
   cd vidrom-intercom
   ```

3. Open `app.json` and update:
   - `"name": "Vidrom Intercom"`
   - `"slug": "vidrom-intercom"`
   - Set `android.package`: `"com.vidrom.ai.intercom"`
   - Set `ios.bundleIdentifier`: `"com.vidrom.ai.intercom"`

4. Replace the content of `App.js` with a single-page UI:
   - Centered View container
   - Text component displaying "Vidrom Intercom"
   - Basic styling (centered text, readable font size)

5. Test on device/emulator:
   ```bash
   npx expo start
   ```
   - Verify on Android emulator or physical device
   - Verify on iOS simulator (if on macOS) or Expo Go on iPhone

## Step 1.2 - Create the "vidrom-home" Expo App

1. From the project root, run:
   ```bash
   npx create-expo-app vidrom-home --template blank
   ```

2. Navigate into the project:
   ```bash
   cd vidrom-home
   ```

3. Open `app.json` and update:
   - `"name": "Vidrom Home"`
   - `"slug": "vidrom-home"`
   - Set `android.package`: `"com.vidrom.ai.home"`
   - Set `ios.bundleIdentifier`: `"com.vidrom.ai.home"`

4. Replace the content of `App.js` with a single-page UI:
   - Centered View container
   - Text component displaying "Vidrom Home"
   - Basic styling (centered text, readable font size)

5. Test on device/emulator:
   ```bash
   npx expo start
   ```
   - Verify on Android emulator or physical device
   - Verify on iOS simulator (if on macOS) or Expo Go on iPhone

## Step 1.3 - Verify Project Structure

After completion, the workspace should look like:

```
vidrom-ai/
├── design/
│   ├── spec.md
│   ├── step1.md
│   ├── step1-steps.txt
│   └── step2.md
├── vidrom-intercom/       <-- Intercom device app (Android SBC)
│   ├── App.js
│   ├── app.json
│   ├── package.json
│   └── ...
└── vidrom-home/           <-- Resident/user mobile app (Android + iOS)
    ├── App.js
    ├── app.json
    ├── package.json
    └── ...
```

## Step 1.4 - Smoke Test Both Apps

1. Run vidrom-intercom and confirm:
   - App launches without errors
   - "Vidrom Intercom" text is visible and centered on screen

2. Run vidrom-home and confirm:
   - App launches without errors
   - "Vidrom Home" text is visible and centered on screen

## Completion Criteria

- ✅ Both Expo projects created and runnable
- ✅ Each app shows a single page with the appropriate label
- ✅ app.json configured with correct names, slugs, and bundle identifiers
- ✅ Both apps testable via Expo Go or emulator on Android and iOS
