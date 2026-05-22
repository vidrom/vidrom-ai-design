# Provisioning And Token Flow

```mermaid
sequenceDiagram
    participant Portal as Admin / Management Portal
    participant Lambda as Portal API Lambda
    participant DB as PostgreSQL
    participant Intercom as Intercom App
    participant Home as Home App
    participant Signal as Signaling HTTP API

    Portal->>Lambda: create / reprovision device
    Lambda->>DB: insert or update intercom + provisioning code
    Lambda-->>Portal: provisioning code

    Intercom->>Signal: POST /api/devices/provision(code)
    Signal->>DB: validate provisioning code
    Signal-->>Intercom: device JWT + deviceId + buildingId
    Intercom->>Intercom: persist token in AsyncStorage
    Intercom->>Signal: WS register(intercom token)

    Home->>Home: authenticate with Firebase auth
    Home->>Signal: POST /api/home/resolve-apartment(email)
    Signal->>DB: lookup user + apartment + building
    Signal-->>Home: apartmentId + userId + building metadata

    Home->>Signal: POST /register-fcm-token
    Home->>Signal: POST /register-voip-token
    Signal->>DB: upsert device_tokens
    Signal->>DB: upsert device_health

    Home->>Signal: WS register(home, apartmentId)
    Signal->>DB: resolve apartment -> building
    Signal->>Signal: map home client to intercom for that building
```

## What This Flow Establishes

- Intercom identity is device-based and comes from provisioning.
- Home identity is user-based and is later mapped to apartment and building scope.
- Push targeting depends on device token rows stored per apartment and user.