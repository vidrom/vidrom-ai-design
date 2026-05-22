# Intercom App Architecture

```mermaid
flowchart TB
    App[App.js] --> Storage[AsyncStorage\ndeviceToken / deviceId / buildingId]
    App --> Setup[DeviceSetupScreen.js]
    App --> Nav[Home / DoorCode / ApartmentList / InCall screens]
    App --> CallMgr[useCallManager.js]

    Setup --> Provision[/api/devices/provision/]
    CallMgr --> WS[Persistent WebSocket\nregister intercom token]
    CallMgr --> WebViewHost[Hidden WebView host]
    CallMgr --> Errors[errorReporter.js]

    WebViewHost --> Html[webrtcHtml.js]
    Html --> WebRTC[Browser WebRTC stack\ncamera + mic + SDP answer]

    ApartmentListScreen[ApartmentListScreen.js] --> Ring[handleRing apartmentId]
    DoorCodeScreen[DoorCodeScreen.js] --> DoorAPI[door / validation APIs]
    InCallScreen[InCallScreen.js] --> Hangup[handleHangUp]

    CallMgr --> Ring
    CallMgr --> Watch[watch / watch-end signaling]
    CallMgr --> OfferBuffer[offer + ICE buffering until WebView ready]
```

## Notes

- The intercom is treated as an always-on kiosk device, so it keeps a persistent signaling socket with reconnect logic.
- WebRTC runs inside a WebView-backed HTML runtime rather than directly through native React Native WebRTC UI.
- The intercom receives offers from the home app and answers them after the WebView initializes camera and microphone capture.