# Home App Architecture

```mermaid
flowchart TB
    App[App.js] --> Auth[authService.js\nFirebase auth\nGoogle / Apple sign-in]
    App --> HomeScreen[HomeScreen.js]

    HomeScreen --> Resolve[/api/home/resolve-apartment/]
    HomeScreen --> CallMgr[useCallManager.js]
    HomeScreen --> FCM[fcmService.js]
    HomeScreen --> VoIP[voipService.js]
    HomeScreen --> CallKit[callkitService.js]
    HomeScreen --> Notify[callNotification.js\nNotifee]

    CallMgr --> WS[On-demand WebSocket\nregister home\naccept / decline / watch]
    CallMgr --> Peer[react-native-webrtc\nRTCPeerConnection]
    CallMgr --> RTC[/api/rtc-config/]
    CallMgr --> Delivery[/api/home/calls/:id/ack/]
    CallMgr --> HttpAccept[HTTP accept reservation]

    FCM --> Token[/register-fcm-token/]
    VoIP --> VoipToken[/register-voip-token/]

    HomeScreen --> Incoming[IncomingCallScreen.js]
    HomeScreen --> Remote[RTCView remote video]

    subgraph Platform Integrations
        FCMCloud[FCM foreground/background]
        PushKit[PushKit]
        NativeCallKit[CallKit]
        NotifeeNative[Android full-screen notification]
    end

    FCM <--> FCMCloud
    VoIP <--> PushKit
    CallKit <--> NativeCallKit
    Notify <--> NotifeeNative
```

## Notes

- The home app delays WebSocket connection until an incoming call, a watch request, or a resumed call path requires it.
- The app bridges three incoming-call surfaces: FCM, VoIP PushKit, and native call UI.
- The home client is the WebRTC offerer for both calls and watch sessions.