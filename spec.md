# Vidrom WebRTC Intercom System - Specification

## Project Overview

- **WebRTC Intercom system**
- Intercom hardware is based on Android SBC
- Client App supports both Android and iOS

## Intercom Device (Android)

- Android-based
- Has a list of residents
- Can call each resident and have WebRTC audio + video calls
- Can use a code to open the door

## User App (Android + iOS)

- Can receive WebRTC calls
- Can push a button to open the door
- Can use a button to watch the intercom camera video while not in a call
- When receiving a call from the intercom, a full screen notification will be shown - similar to WhatsApp incoming calls
- Login via Google or Apple accounts

## Technology Stack

- **WebRTC** for audio and video calls

## Key Features

- Google Login
- Apple Login
- Full screen notification
- Audio and video WebRTC calls

## Signaling Contract

The current WebSocket signaling contract is defined by the live implementation in
the home app, intercom app, and signaling server.

- Canonical message names: `ring`, `offer`, `answer`, `candidate`, `accept`, `decline`, `hangup`, `watch`, `watch-end`
- The home app is the SDP offerer for both calls and watch sessions
- The intercom app responds with the SDP answer
- Older labels such as `ice-candidate`, `end-call`, `watch-start`, and `watch-stop` are not the live contract
