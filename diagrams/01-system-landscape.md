# System Landscape

```mermaid
flowchart LR
    subgraph Clients
        Home[Home App\nReact Native\niOS + Android]
        Intercom[Intercom App\nReact Native\nAndroid SBC]
        Admin[Admin Portal\nadmin.html]
        Mgmt[Management Portal\nmanagement.html]
    end

    subgraph Edge
        PortalEdge[portal.vidrom.com\nCloudFront]
        SignalEdge[signaling.vidrom.com\nALB + TLS]
    end

    subgraph Compute
        Api[Portal API Lambda\nadmin + management REST]
        Signal[Signaling EC2 Service\nHTTP + WebSocket]
        Twilio[Twilio NTS\nmanaged STUN/TURN]
    end

    subgraph Data
        RDS[(PostgreSQL RDS)]
        Secrets[Secrets Manager]
        Bucket[S3 Portal Assets]
    end

    subgraph Push
        FCM[Firebase Cloud Messaging]
        APNs[APNs VoIP Push]
    end

    Admin --> PortalEdge
    Mgmt --> PortalEdge
    PortalEdge --> Bucket
    PortalEdge --> Api

    Home -->|HTTPS| SignalEdge
    Home -->|WSS on demand| SignalEdge
    Intercom -->|HTTPS| SignalEdge
    Intercom -->|Persistent WSS| SignalEdge
    SignalEdge --> Signal

    Signal --> RDS
    Api --> RDS
    Signal --> Secrets
    Api --> Secrets
    Signal --> FCM
    Signal --> APNs
    Home -. receives pushes .- FCM
    Home -. iOS VoIP .- APNs
    Home -->|ICE config| Signal
    Intercom -->|ICE config| Signal
    Signal -->|short-lived ICE config| Twilio
    Home <-->|WebRTC media| Twilio
    Intercom <-->|WebRTC media| Twilio
```

## Notes

- The home app keeps signaling mostly on demand, while the intercom stays persistently connected.
- Portal HTML is static and served from S3 through CloudFront, but the admin and management APIs run in Lambda.
- The EC2 signaling service still owns both WebSocket signaling and a set of stateful HTTP endpoints used by the mobile apps.
- TURN relay is now provided by Twilio NTS rather than a coturn process on the signaling host.