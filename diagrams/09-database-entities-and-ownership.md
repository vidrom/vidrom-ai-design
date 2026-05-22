# Database Entities And Ownership Boundaries

```mermaid
flowchart TB
    subgraph Core[Core domain entities]
        Buildings[(buildings)]
        Apartments[(apartments)]
        Users[(users)]
        Intercoms[(intercoms)]
        Notifications[(notifications)]
        Settings[(global_settings)]
    end

    subgraph Access[Ownership and scope boundaries]
        AptResidents[(apartment_residents)]
        BldgManagers[(building_managers)]
    end

    subgraph Runtime[Operational and audit records]
        Calls[(calls)]
        Audit[(audit_logs)]
        DeviceTokens[(device_tokens)]
        DeliveryAcks[(call_delivery_acks)]
        DeviceHealth[(device_health)]
        ClientErrors[(client_errors)]
    end

    Buildings -->|1 to many| Apartments
    Buildings -->|1 to many| Intercoms
    Buildings -->|1 to many| Notifications

    Apartments -->|many to many via join| AptResidents
    Users -->|many to many via join| AptResidents

    Buildings -->|many to many via join| BldgManagers
    Users -->|many to many via join| BldgManagers

    Buildings -->|scopes| Calls
    Apartments -->|target of| Calls
    Intercoms -->|origin of| Calls

    Buildings -->|context| Audit
    Apartments -->|context| Audit
    Users -->|context| Audit
    Intercoms -->|context| Audit
    Calls -->|context| Audit

    Apartments -->|token delivery scope| DeviceTokens
    Users -->|token owner| DeviceTokens

    Calls -->|delivery telemetry| DeliveryAcks
    Users -->|ack actor| DeliveryAcks

    Apartments -->|health scope| DeviceHealth
    Users -->|device owner| DeviceHealth

    Buildings -->|optional context| ClientErrors
    Apartments -->|optional context| ClientErrors
    Users -->|optional context| ClientErrors
    Intercoms -->|optional context| ClientErrors

    classDef access fill:#f6efe1,stroke:#8a5a00,color:#2d1c00
    classDef core fill:#edf6ff,stroke:#1f5d99,color:#10243d
    classDef runtime fill:#eef8ee,stroke:#2c6b2f,color:#18361a

    class Buildings,Apartments,Users,Intercoms,Notifications,Settings core
    class AptResidents,BldgManagers access
    class Calls,Audit,DeviceTokens,DeliveryAcks,DeviceHealth,ClientErrors runtime
```

## Ownership Model

- `admin` users are global operators. Their access is role-based and not constrained by join tables.
- `manager` users are constrained by `building_managers`, which defines which buildings they can administer.
- `resident` users are constrained by `apartment_residents`, which defines which apartments and therefore which building-scoped calls and notifications they can access.
- `intercom` devices are constrained by `intercoms.building_id`, which binds each device to one building.

## Boundary Notes

- `apartment_residents` is the main resident authorization boundary for the home app.
- `building_managers` is the management-portal authorization boundary.
- `device_tokens` and `device_health` are not standalone identity sources. They must stay subordinate to the owning `users` and `apartments` records.
- `calls`, `audit_logs`, and `call_delivery_acks` are operational records whose visibility should be derived from apartment or building ownership, not from caller-supplied IDs.