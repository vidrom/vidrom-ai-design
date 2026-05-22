# Incoming Call Flow

```mermaid
sequenceDiagram
    participant Intercom as Intercom App
    participant Server as Signaling Server
    participant DB as PostgreSQL
    participant Push as FCM / APNs
    participant Home as Home App
    participant Native as CallKit / Notifee

    Intercom->>Server: WS ring(apartmentId)
    Server->>DB: insert calls(status=calling) + audit log
    Server->>Server: activeCall.start + pendingRing timeout
    Server-->>Home: WS ring to connected apartment clients
    Server->>Push: fan out incoming-call pushes
    Push-->>Home: incoming-call payload
    Home->>Native: show incoming UI
    Home->>Server: HTTP delivery ack events
    Home->>Server: connect WS as home
    Server-->>Home: re-send pending ring if needed

    alt First resident accepts
        Home->>Server: HTTP accept reservation
        Server->>Server: reserve winner for callId
        Home->>Server: WS accept(callId, userId)
        Server->>DB: update calls(status=accepted) + audit log
        Server-->>Intercom: accept
        Server-->>Home: call-taken to all other apartment clients
        Server->>Push: call-taken pushes for non-WS devices
        Home->>Server: offer
        Server-->>Intercom: offer
        Intercom->>Server: answer
        Server-->>Home: answer
        Home->>Server: ICE candidates
        Intercom->>Server: ICE candidates
        Home-->>Intercom: WebRTC media established
    else Nobody accepts before timeout
        Server->>DB: update calls(status=unanswered)
        Server-->>Intercom: apartment-unavailable or timeout outcome
    end

    opt Door opened or either side hangs up
        Home->>Server: open-door or hangup
        Intercom->>Server: hangup
        Server->>DB: update calls(status=ended) + audit log
        Server-->>Other side: hangup
        Server->>Server: clear activeCall + timers + retries
    end
```

## Important Rules

- Call delivery is apartment-scoped, not single-device scoped.
- First accept wins. Every non-winning device must receive `call-taken` quickly.
- The home app is the SDP offerer.