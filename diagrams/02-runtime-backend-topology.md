# Runtime Backend Topology

```mermaid
flowchart TB
    ALB[ALB / signaling.vidrom.com] --> Server

    subgraph Server[EC2 signaling runtime]
        Http[httpRoutes.js\nstateful HTTP endpoints]
        Ws[wsHandler.js\nWebSocket signaling]
        State[connectionState.js\nintercoms\nhomeClients\nactiveCalls\npendingRings]
        Retry[retryOrchestrator.js\npush retry loop]
        RingTimeout[ringTimeout.js\nper-building/apartment timeout]
        APNsSvc[apnsService.js]
        Startup[startupConfig.js\nFirebase + APNs + TURN config]
        Auth[auth.js + devices.js\nJWT + device validation]
        Health[deviceHealthScore.js\ncall delivery health]
    end

    Http <--> State
    Ws <--> State
    Ws --> Retry
    Ws --> RingTimeout
    Http --> Health
    Ws --> Health
    Ws --> APNsSvc
    Ws --> Auth
    Http --> Auth
    Startup --> Http
    Startup --> Ws

    Server --> DB[(PostgreSQL)]
    Ws --> FCM[Firebase Admin]
    APNsSvc --> APNs[APNs]

    DB --> Calls[calls\naudit_logs\ncall_delivery_attempts\ncall_delivery_acks]
    DB --> Devices[intercoms\ndevice_tokens\ndevice_health]
    DB --> Model[buildings\napartments\nusers\napartment_residents\nbuilding_managers]
```

## Responsibilities

- `wsHandler.js` owns signaling, ring fanout, first-accept-wins, watch mode, and push triggers.
- `httpRoutes.js` owns token registration, apartment resolution, device provisioning, delivery acknowledgements, and RTC config.
- `connectionState.js` is the in-memory coordination layer that makes the backend stateful.
- RDS stores the durable model and delivery history, while active call routing still depends on process memory.